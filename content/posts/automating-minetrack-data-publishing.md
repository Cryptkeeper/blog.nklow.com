---
title: "Automating Data Publishing with BunnyCDN"
date: 2020-08-02T15:35:24-05:00
tags: ["minecraft", "minetrack", "sqlite", "bunnycdn"]
summary: "Automatically publishing daily Minetrack database replicas using BunnyCDN"
---

As [Minetrack](https://minetrack.me) continually polls Minecraft servers, the  collected player counts are timestamped and inserted into an [SQLite](https://sqlite.org) database. Never one to hoard data (...and to free up some disk space) I began publishing Minetrack's databases at [data.minetrack.me](https://data.minetrack.me). 

Unfortunately the process of preparing a database for publishing is rather annoying.

1. `database.sql` is locked while Minetrack is running. Minetrack must be temporarily halted so the database file can be safely duplicated.
2. Determine the start and end timestamps required. Prune the database file of any records outside those bounds.
3. [`VACUUM`](https://www.sqlite.org/lang_vacuum.html) the database file to repack its structure given the large amount of previously pruned data.
4. zip the database file and upload to [BunnyCDN](https://bunnycdn.com). Once uploaded, add the link and information to [data.minetrack.me](https://data.minetrack.me).
5. Tweet about it.

Worse yet, this manual process results in an unreliable, and often delayed, publishing schedule. While failing to quickly publish Minecraft server player counts is unlikely to upend the the world, it is another vexing aspect of an already painful process.

Obviously this process is ripe for automation via a hacky shell script.

### Quick & Dirty Database Replication
Starting in [Minetrack 5.5.8](https://github.com/Cryptkeeper/Minetrack/blob/master/docs/CHANGELOG.md), setting `createDailyDatabaseCopy: true` in `config.json` will automatically create a daily replica of `database.sql`, formatted as `database_copy_%d-%m-%yyyy.sql`.

_While I would prefer to see this operation happening external to the program, enabling Minetrack to create the database replica at a high level eases the complexity of producing a "publish ready" database whilst reducing potential failure points._

With Minetrack replicating the database as it runs, with support for date rollovers, we need a script to:

1. zip the database file.
2. Upload the compressed database file to [BunnyCDN](https://bunnycdn.com) for distribution.
3. Cleanup the daily rollover file. (A full copy of the historical data is retained in the primary `database.sql` file.)

With these requirements in mind, its easy to glue together a quick shell script.

```bash
#!/bin/bash

upload_db() {
	local databaseFile=$1
	local databaseZipFile="${databaseFile}.zip.temp"
	
	zip $databaseZipFile $databaseFile
	
	# TODO: upload file...
	
	rm $databaseZipFile
}
```

Our `upload_db` function will accept a parameter, named `databaseFile`. This is the database file that will be compressed and eventually uploaded. 

The provided database file is zipped into a newly named file. For example, `mydatabase.sql` becomes `mydatabase.sql.zip.temp`. This attempts to produce a unique-_ish_ name to prevent potential clashes if multiple copies of the shell script are executed at the same time.

### Uploading to BunnyCDN
To upload the newly created zip file, [BunnyCDN](https://bunnycdn.com) luckily offers a simple [storage API](https://bunnycdnstorage.docs.apiary.io/#) with a route specifically for [uploading new files](https://bunnycdnstorage.docs.apiary.io/#reference/0/storagezonenamepathfilename/put) using a `PUT` request. We can use [curl](https://curl.haxx.se/) to make the HTTP request.

```bash
#!/bin/bash

BUNNYCDN_API_KEY=""
BUNNYCDN_REGION="ny"
BUNNYCDN_STORAGE_ZONE="minetrackdata"

upload_db() {
	local databaseFile=$1
	local databaseZipFile="${databaseFile}.zip.temp"
	
	zip $databaseZipFile $databaseFile
	
	local uploadPath=$2
	local uploadFile=$3
	
	curl --upload-file $databaseZipFile \
		--header "AccessKey: ${BUNNYCDN_API_KEY}" \
		"https://${BUNNYCDN_REGION}.storage.bunnycdn.com/${BUNNYCDN_STORAGE_ZONE}/${uploadPath}/${uploadFile}"
		
	rm $databaseZipFile
}
```

In order to use BunnyCDN's API, there is a few values we need to set.

1. In the [Authentication and Authorization](https://bunnycdnstorage.docs.apiary.io/#authentication) section of the documentation, it describes a required "AccessKey" header. You will need to locate your API key and set `BUNNYCDN_API_KEY` to match.
2. If your storage zone is not in BunnyCDN's "ny" region, you will need to set `BUNNYCDN_REGION` to match.
3. `BUNNYCDN_STORAGE_ZONE` is the name of your storage zone. In my case, this is "minetrackdata". If you are using a `b-cdn.net` domain, such as `minetrackdata.b-cdn.net`, the subdomain is your storage zone name.

Finally, two new parameters, `uploadPath` and `uploadFile` have been added. Together, these control where the uploaded file is stored in your BunnyCDN storage zone.

### Customizing the Upload Script
To better integrate this function with my expected use case, I'm going to make a few changes.

1. Using [cron](https://en.wikipedia.org/wiki/Cron), I have scheduled the script to run at 12:05am each day - 5 minutes after Minetrack creates the _next_ daily replica database file. Because it is running "the next day", the script will need to use yesterday's date.
1. `uploadFile` is now automatically set using yesterday's date. 
2. `databaseFile` is now also deleted after being uploaded.

```bash
#!/bin/bash

BUNNYCDN_API_KEY=""
BUNNYCDN_REGION="ny"
BUNNYCDN_STORAGE_ZONE="minetrackdata"

YESTERDAY_DATE=$(date +%-d-%-m-%Y -d "1 day ago")

upload_db() {
	local databaseFile=$1
	local databaseZipFile="${databaseFile}.zip.temp"
	
	zip $databaseZipFile $databaseFile
	
	local uploadPath=$2
	local uploadFile="${YESTERDAY_DATE}.sql.zip"

	curl --upload-file $databaseZipFile \
		--header "AccessKey: ${BUNNYCDN_API_KEY}" \
		"https://${BUNNYCDN_REGION}.storage.bunnycdn.com/${BUNNYCDN_STORAGE_ZONE}/${uploadPath}/${uploadFile}"
		
	rm $databaseZipFile
	rm $databaseFile
}
```

_There's an obvious lack of error handling in this script. As such, you should NOT use this for any important purposes. My usage of the script is with duplicated data that can be safely lost, with monitoring present to detect failures._

Finally, the script can call the `upload_db` function and we're done.

```bash
upload_db "Minetrack/database_copy_${YESTERDAY_DATE}.sql" "Daily"
```

This automatically uploads yesterday's database file from the `Minetrack/` directory into the `Daily/` directory on the storage zone. Due to my changes to `upload_db`, the file will automatically be named after yesterday's date.

If we manually construct the file path, we should find our newly uploaded file.

`https://minetrackdata.b-cdn.net/Daily/1-8-2020.sql.zip`

{{< figure src="/MinetrackData_images/download_preview.png" >}}

Voila!

Our daily Minetrack database replica is now set to be automatically uploaded to [BunnyCDN](https://bunnycdn.com) for cheap distribution to fans of numbers all around the world.

Using BunnyCDN's [storage API](https://bunnycdnstorage.docs.apiary.io/#reference/0//{storagezonename}/{path}/), we can further develop the script to automatically generate a page of download links - but that's for next time.