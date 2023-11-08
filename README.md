# nuget-api-dataget
Return an xml or json response from NuGet Api (APIv2, v3) by using NuGet owner's name and stores the result in a predefined Gist in Base64 Data Format. More info is displayed in the 'Main Workflow File' example below.


## Inputs

|Name|Type|Description|
|---|---|---|
|`ownerName`|string|The ownername of the nuget package registry to retrieve data from|
|`isXmlResp`|string|In 'secrets' of the repository you intend to run this on, set only value of TRUE or FALSE|
|`GIST_ID`|string|The ID of the predefined Gist to use when updating Gist content with the retrieved data|
|`ACTION_TOKEN`|string|In 'secrets' of the repository you intend to run this on, the GitHub API Token|

<br/>

### About - Access tokens
To run this action the access token must have scope `admin:org` on PAT  or `owner:repo` access levels.


## Outputs

|Name|Type|Description|
|---|---|---|
|`Raw API v2 XML Nuget Entries Response`|string|A Base64-Encoded string of xml response for NuGet v2 API listed packages|
|`Raw API v3 Json Nuget Entries Response`|string|A Base64-Encoded string of json response for NuGet v3 API listed packages|


<br/><br/>

### __Note: The response requires a Gist for the response to be stored appropriately. Action is run at end of each week in the example.__


<br/>



## How it works
Main Workflow File:
``` yaml
name: Weekly Content Updates

on:
  schedule:
  - cron: "0 14 * * 0"

jobs:
  weeklycontentupdate:
    runs-on: ubuntu-latest
    steps:
    - name: Set updated content (XML, JSON)
      id: setUpdatedContent
      env:
        ownerName: your_nuget_username
        isXmlResp: ${{ secrets.IS_XML }}
      run: |
        if [ ${{ env.isXmlResp }} == 'TRUE' ]; then
          echo "NUG_RESPONSE<<EOF" >> $GITHUB_ENV
          echo $(curl --no-progress-meter 'https://www.nuget.org/api/v2/Search()?$filter=IsLatestVersion&searchTerm=%27owner%3A${{ env.ownerName }}%27&targetFramework=%27%27&includePrerelease=false&$skip=0&semVerLevel=2.0.0' | base64) | sed 's/ //g' >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV
        else
          echo "NUG_RESPONSE<<EOF" >> $GITHUB_ENV
          echo $(curl --no-progress-meter 'https://api-v2v3search-0.nuget.org/query?q=${{ env.ownerName }}&prerelease=false' | base64) | sed 's/ //g' >> $GITHUB_ENV
          echo "EOF" >> $GITHUB_ENV
        fi
    - name: Update accesible Gists
      id: updateGists
      env:
        GIST_ID: your_predefined_gist_id
        ACTION_TOKEN: ${{ secrets.GH_ACTION }}
        RESPONSE: "${{ env.NUG_RESPONSE }}"
      run: |
        curl --no-progress-meter  -L -X PATCH -H "Accept: application/vnd.github+json" -H "Authorization: Bearer $ACTION_TOKEN" -H "X-Github-Api-Version: 2022-11-28" https://api.github.com/gists/${{ env.GIST_ID }} -d '{"description":"rssdata","files":{"test_gst.txt":{"content":"${{ env.RESPONSE }}" }}}'
```