---
title: "Automating Data Publishing with BunnyCDN (Part 2)"
date: 2020-08-11T16:53:22-05:00
tags: ["minecraft", "minetrack", "sqlite", "bunnycdn"]
summary: "Automatically publishing daily Minetrack database replicas using BunnyCDN"
---

This post is a continuation of ["Automating Data Publishing with BunnyCDN"]({{< ref "posts/automating-minetrack-data-publishing.md" >}}).

With a quick shell script to automatically upload daily copies of [Minetrack's](https://minetrack.me) database to [BunnyCDN](https://bunnycdn.com) for publishing in hand, its time to generate a page of download links.

Since I already have a previously designed page, I'm going to insert some unique strings to use as substitution patterns for [sed](https://en.wikipedia.org/wiki/Sed) later on. This will be saved as `index.html.template`. You're free to name it as you wish, but you must remember to update any usages of the filename.

```html
<details>
	<summary>Minetrack Bedrock</summary>
	___BUILD_SCRIPT_ENTRY_Bedrock___
</details>
<details>
	<summary>Minetrack Java</summary>
	___BUILD_SCRIPT_ENTRY_Java___
</details>
```

### Using the BunnyCDN Storage API
Looking back at BunnyCDN's [storage API](https://bunnycdnstorage.docs.apiary.io/#) we can see a route which "retrieves a list of files and directories located in the given directory". To help keep things organized, I'm going to create a new shell script, `build_website.sh`.

```bash
#!/bin/bash

BUNNYCDN_API_KEY=""
BUNNYCDN_REGION="ny"
BUNNYCDN_STORAGE_ZONE="minetrackdata"
```

Same as previous usage of the BunnyCDN storage API, there is a few values we need to set.

1. In the [Authentication and Authorization](https://bunnycdnstorage.docs.apiary.io/#authentication) section of the documentation, it describes a required "AccessKey" header. You will need to locate your API key and set `BUNNYCDN_API_KEY` to match. 
2. If your storage zone is not in BunnyCDN's "ny" region, you will need to set `BUNNYCDN_REGION` to match.
3. `BUNNYCDN_STORAGE_ZONE` is the name of your storage zone. In my case, this is "minetrackdata". If you are using a `b-cdn.net` domain, such as `minetrackdata.b-cdn.net`, the subdomain is your storage zone name.

Since this script's usage of the API doesn't modify anything, you should considering using the "read only" API key offered by BunnyCDN.

```bash
#!/bin/bash

BUNNYCDN_API_KEY=""
BUNNYCDN_REGION="ny"
BUNNYCDN_STORAGE_ZONE="minetrackdata"

render_table() {
	local BUNNYCDN_PATH=$1
	local BUNNYCDN_FILE_LIST=$(curl --header "AccessKey: ${BUNNYCDN_API_KEY}" --silent "https://${BUNNYCDN_REGION}.storage.bunnycdn.com/${BUNNYCDN_STORAGE_ZONE}/${BUNNYCDN_PATH}/")
	
	echo $BUNNYCDN_FILE_LIST
}
```

Since Minetrack publishes its files to two pre-determined directories, I've opted to make the `BUNNYCDN_PATH` variable a function parameter instead of hardcoding its value.

Same as before, we are using [curl](https://curl.haxx.se/) to make the API request. This time around I've added the `--silent` argument to avoid curl outputting any text. This is important because the output will now be fed into [jq](https://stedolan.github.io/jq/), a tool for parsing JSON.

### JSON Parsing with jq
If you call `render_table`, you'll now see a JSON formatted array like so. I've removed several fields in my example (such as `LastChanged` and `IsDirectory`) since I won't be using them.
```json
[
  {
    "ObjectName": "1-8-2020.sql.zip",
    "Length": 303293,
    "DateCreated": "2020-08-02T01:49:17.17",
  },
  ...
]
```

By feeding `$BUNNYCDN_FILE_LIST` into [jq](https://stedolan.github.io/jq/), we can easily iterate over the JSON array to handle each file individually. Additionally, two important arguments are being passed to `jq`.

1. `--compact-output` prints each JSON object without newlines. This will allow us to parse each JSON object individually within the `for` loop.
2. `sort_by(.DateCreated)` sorts (in ascending order) the JSON array using the `DateCreated` field within each JSON object. This avoids the BunnyCDN API returning the JSON objects out of order.

For more information on `jq` arguments and usage, check out its [manual](https://stedolan.github.io/jq/manual/).

```bash
#!/bin/bash

BUNNYCDN_API_KEY="d8ec9476-16ff-409e-bea303a7aae6-3505-4355"
BUNNYCDN_REGION="ny"
BUNNYCDN_STORAGE_ZONE="minetrackdata"

render_table() {
	local BUNNYCDN_PATH=$1
	local BUNNYCDN_FILE_LIST=$(curl --header "AccessKey: ${BUNNYCDN_API_KEY}" --silent "https://${BUNNYCDN_REGION}.storage.bunnycdn.com/${BUNNYCDN_STORAGE_ZONE}/${BUNNYCDN_PATH}/")

	for FILE in $(echo $BUNNYCDN_FILE_LIST | jq --compact-output 'sort_by(.DateCreated) | .[]')
	do
		# ...
	done
}
```

At this point, in each iteration `$FILE` is a JSON object representing an individual file in the BunnyCDN storage zone, in ascending order according to its `DateCreated` field.

```json
{
	"ObjectName": "1-8-2020.sql.zip",
	"Length": 303293,
	"DateCreated": "2020-08-02T01:49:17.17"
}
```

_If you have subdirectories in your path, you may wish to check the `IsDirectory` field to prevent your script from including subdirectories._

Using `jq` again, we can extract whatever JSON object fields we wish into variables in our script:

```bash
myScriptVariable=$(echo $FILE | jq --raw-output '.MyJsonVariable')
```

The `--raw-output` argument (also available as `-r`) ensures `jq` outputs the value directly, without JSON formatting. For example, `.ObjectName` will print as `1-8-2020.sql.zip` and not its JSON formatted equivalent, `"1-8-2020.sql.zip"` (note the double quotes).

I've gone ahead and opted to further format the `lengthMegabytes` and `dateCreated` fields to make them human friendly.

```bash
for FILE in $(echo $BUNNYCDN_FILE_LIST | jq --compact-output 'sort_by(.DateCreated) | .[]')
do
	# .Length is in bytes
	# Divide by 1024^2 to convert it into megabytes
	# Use awk's printf to format it with 2 decimal points of precision
	lengthMegabytes=$(echo $FILE | jq -r '.Length' | awk '{printf "%.2f", $0 / 1024 / 1024}')
	
	# .DateCreated is a date+time string (2020-08-02T01:49:17.17)
	# Use cut to split the string at 'T' and return the 1st part
	# 2020-08-02T01:49:17.17 -> 2020-08-02
	dateCreated=$(echo $FILE | jq -r '.DateCreated' | cut -d 'T' -f 1)
	
	# Use the value directly, no formatting changes needed
	objectName=$(echo $FILE | jq -r '.ObjectName')
done
```

At this point we've established how to make the BunnyCDN API request, parse the JSON array response, read each JSON object and extract the values we're interested in. All that's left is to generate some HTML with our extracted values, and insert it into the previously created `index.html.template` file.

### Generating HTML
```bash
render_table() {
	local BUNNYCDN_PATH=$1
	local BUNNYCDN_FILE_LIST=$(curl --header "AccessKey: ${BUNNYCDN_API_KEY}" --silent "https://${BUNNYCDN_REGION}.storage.bunnycdn.com/${BUNNYCDN_STORAGE_ZONE}/${BUNNYCDN_PATH}/")

	for FILE in $(echo $BUNNYCDN_FILE_LIST | jq -c 'sort_by(.DateCreated) | .[]')
	do
		lengthMegabytes=$(echo $FILE | jq -r '.Length' | awk '{printf "%.2f", $0 / 1024 / 1024}')
		dateCreated=$(echo $FILE | jq -r '.DateCreated' | cut -d 'T' -f 1)
		objectName=$(echo $FILE | jq -r '.ObjectName')
		
		local rows="$rows\
			<tr> \
				<td>${objectName}</td>\
				<td>${lengthMegabytes} MB</td>\
				<td>${dateCreated}</td>\
				<td><a href=\"https://${BUNNYCDN_STORAGE_ZONE}.b-cdn.net/${BUNNYCDN_PATH}/${objectName}\">BunnyCDN</a></td>\
			</tr>"
	done

	echo "<table>\
			<thead>\
				<tr>\
					<td>File</td>\
					<td>Size</td>\
					<td>Date Uploaded</td>\
					<td>Download</td>\
				</tr>\
			</thead>\
			<tbody>\
				${rows}\
			</tbody>\
		</table>"
}
```

Using a local variable, `$rows`, each iteration of the `for` loop now appends a HTML table row containing our extracted variables. At the end of the `render_table` function, it is inserted into a pre-defined HTML table string.

While BunnyCDN's storage API returns a `Path` value for each file, it includes the storage zone name (for example, `/minetrackdata/Test/` instead of `Test/`). This is an issue since download links only use the storage zone as the subdomain (assuming you're not using a custom domain), and not part of the path. To side step this issue, I've opted to manually construct the download link URL by re-using the previously created `$BUNNYCDN_PATH` variable.

```bash
https://${BUNNYCDN_STORAGE_ZONE}.b-cdn.net/${BUNNYCDN_PATH}/${objectName}
```

Upon calling `render_table`, a HTML table structure should now be printed.

```html
<table> 
	<thead>
		<tr>
			<td>File</td>
			<td>Size</td>
			<td>Date Uploaded</td>
			<td>Download</td>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>1-8-2020.sql.zip</td>
			<td>0.29 MB</td>
			<td>2020-08-02</td>
			<td><a href="https://minetrackdata.b-cdn.net/Bedrock/1-8-2020.sql.zip">BunnyCDN</a></td>
		</tr>
		<!-- ... -->
	</tbody>
</table>
```

### String Substitution with sed
Finally, using [sed](https://en.wikipedia.org/wiki/Sed) we can substitute our previously inserted unique marker strings with our generated HTML.

Since I'm generating several tables (with a strong possibility of adding more), I've added an array of strings. Each string matches both its path in the BunnyCDN storage zone, and its unique marker string in the template.

```bash
TEMPLATE=$(cat "index.html.template")

TABLES=( "Bedrock" "Java" )
for TABLE in "${TABLES[@]}"
do
	HTML=$(render_table $TABLE)
	PATTERN="___BUILD_SCRIPT_ENTRY_${TABLE}___"
	TEMPLATE=$(echo $TEMPLATE | sed "s@${PATTERN}@${HTML}@")
done
```

The `sed` pattern uses `@` instead of the more commonly seen `/` to avoid parsing issues with the `/` characters contained within the generated HTML. `@` is only a suitable replacement since it is not contained within the generated HTML. If your generated HTML contains `@`, you will need to replace its usage.

Finally, `$TEMPLATE` contains our template file with the newly generated and inserted HTML tables. We can output it to a file like so. This will also automatically overwrite the file if it already exists.

```bash 
echo $TEMPLATE > "index.html"
```

### Future Improvements
In my usage I've opted to output the file to `/usr/share/nginx/html/index.html` so that it is served by [nginx](https://www.nginx.com/). I've added the script to my [cron](https://en.wikipedia.org/wiki/Cron) schedule to automatically execute the script after the previous script has files have been uploaded.

Obviously there's room for improvement in these scripts, especially in regards to error handling. That said, these two quick scripts have automated a painful amount of work and are simple enough to easily manage well into the future without confusion. For now, that's enough.
