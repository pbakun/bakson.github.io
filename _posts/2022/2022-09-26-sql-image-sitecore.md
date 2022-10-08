---
layout: post
title: SQL Server 2019 docker image for Sitecore
tags: [docker, sql, Sitecore]
excerpt_separator: <!--more-->
---

Using SQL Server in docker may not be the best idea for production environment, but is certainly useful for development. Docker makes developer's live nice and easy (most times).

<!--more-->

This article was inspired by Sitecore lack of official SQL Server 2019 docker images for development, although it could be useful to anyone dealing with MS SQL with windows containers. Before Sitecore 10+, if you wanted to use docker, you were forced to build your own images from community supported [repository](https://github.com/Sitecore/docker-images). When in need, or just out of curiosity, one could easily customize what are used for based images, entrypoints, scripts run in the meantime. From versions 10+ Sitecore provides officially supported images, which is awesome, but also makes customization a bit harder.

### Root cause
When my team upgraded from Sitecore 9.2 to 9.3 we decided to go with docker images for development. There are many advantages of that approach, although it required some extra work on our side. One of the first issues was SQL database version used. We used our on&#8209;premise development setup with newest (at that time) SQL Server, but with mentioned earlier repo we could only build 2017 image. The easiest would be to go with it, but if you ever tried to downgrade your database you probably not it's pretty much impossible (if I'm wrong please let me know).

### To the point
As said, Sitecore doesn't share how images are built. Luckily with docker we can always access container and check out what sits inside. First step is to find how to build pure MSSQL2019 image. That microsoft gives us for [free](https://github.com/microsoft/mssql-docker). Next step is to pull and inspect Sitecore provided MSSQL2017 image. Using:

```docker
docker pull scr.sitecore.com/sxp/nonproduction/mssql-developer:2017-10.2-ltsc2019
docker image inspect scr.sitecore.com/sxp/nonproduction/mssql-developer:2017-10.2-ltsc2019
```

Given result is an json-looking object containing basic image information, among which we can find docker **entrypoint**. In this case it is a command. It is not exactly the same as entrypoint, but I'm not going to explain it here. Further reading on this topic you can find [here](https://phoenixnap.com/kb/docker-cmd-vs-entrypoint). The only thing you need to know is that running this will spin up container's main process, in our case an SQL server.

```json
"Cmd": ["powershell",
        "-Command",
        "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';",
        "#(nop) ",
        "CMD [\"powershell.exe\" \".\\\\start.ps1\" \"-sa_password $env:SA_PASSWORD\" \"-ACCEPT_EULA $env:ACCEPT_EULA\" \"-attach_dbs \\\"$env:attach_dbs\\\"\" \"-DataDirectory $env:DATA_PATH\" \"-Verbose\"]"
```

If we compare this command with the one from [Dockerfile](https://github.com/microsoft/mssql-docker/blob/master/windows/mssql-server-windows-developer/dockerfile_1) in mentioned repo we can see that they look really similar:

```dockerfile
CMD .\start -sa_password $env:sa_password -ACCEPT_EULA $env:ACCEPT_EULA -attach_dbs \"$env:attach_dbs\" -Verbose
```

Both commands run powershell script `start.ps1` and pass couple environment variables. Although Sitecore's image have one more - `DataDirectory`. That directory will be read in that script and attach all `.mdf` and `.ldf` files into SQL server on container start. Those are files for each database's data and logs. Looks like the only thing we need to do is to replace original `start.ps1` with the one from Sitecore's image and add additional evnironment variable. To make things clear let's have a glance at `docker-compose` example:

```yml
services:
  mssql:
    image: scr.sitecore.com/sxp/nonproduction/mssql-developer:2017-10.2-ltsc2019
    volumes:
      - "./volume:C:\\data" # Folder volume contains databases
    environment:
      SA_PASSWORD: "Password12345"
      ACCEPT_EULA: "Y"
      DATA_PATH: "C:\\data" # Same folder passed to container so script knows where to look to databases
    ports:
      - "14330:1433"
```

Considering copying files from windows container to host system doesn't work too good I will use an awesome VS Code [extension](https://github.com/microsoft/vscode-docker) to get into Sitecore's filesystem and extract its `start.ps1` script. Let's run our docker-compose.yml.

```
docker-compose up -d
```
And using VS Code extension check out how entrypoint script looks like:
{% include center-img.html images="articles/2022/sql-image-sitecore/Code_lHEISkOajS.png" column=1 %}

If you compare `start.ps1` from within the container and the one from [Microsoft's repo](https://github.com/microsoft/mssql-docker/blob/master/windows/mssql-server-windows-developer/start.ps1) you will see that this is the only difference:

```powershell
#Attach databases in data directory
Get-ChildItem -Path $DataDirectory -Filter "*.mdf" | ForEach-Object {
    $databaseName = $_.BaseName.Replace("_Primary", "")
    $mdfPath = $_.FullName

    $primaryDbEnding = $_.Name.Replace(".mdf", ".ldf")
    $logDbEnding = $databaseName + "_log.ldf"

    $ldfPath = Get-ChildItem -Path $DataDirectory | Where-Object {$_.Name -eq $primaryDbEnding -or $_.Name -eq $logDbEnding}
    $ldfPath = $ldfPath.FullName
    $sqlcmd = "IF EXISTS (SELECT 1 FROM SYS.DATABASES WHERE NAME = '$databaseName') BEGIN EXEC sp_detach_db [$databaseName] END;CREATE DATABASE [$databaseName] ON (FILENAME = N'$mdfPath'), (FILENAME = N'$ldfPath') FOR ATTACH;"

    Write-Host "INFO: Attaching '$databaseName'..."

    & sqlcmd -Q $sqlcmd
}
```

Let's build our own MSSQL2019 image using `dockerfile_1` (at the time of writing this article `dockerfile` downloads installation files for MSSQL2017) from repo and replace `start.ps1` with Sitecore's image start script content.
Note that you need to download installation files:
1. SQLServer2019-DEV-x64-ENU.exe
2. SQLServer2019-DEV-x64-ENU.box

```docker
docker build -t sitecore-mssql-developer-2019 -f dockerfile_1 .
```
The build will take a while, but when it'll finish replace image name to the newly built one. If you have any database files you could put them in `volume` folder, otherwise you have to trust my word that as soon as container starts database will attach.

There are plenty of tools you could use to connect to database, eg. SSMS or another [VS Code extension](https://github.com/Microsoft/vscode-mssql). I will stick with VS Code here. Using below connection string I'm connecting to server and bum... My database is attached.
```
Data Source=tcp:localhost,14330;User ID=sa;Password=Password12345;
```
{% include aligner.html images="articles/2022/sql-image-sitecore/Code_YrLQHAHRTB.png,articles/2022/sql-image-sitecore/Code_OxhjAa1Ti8.png" %}

Looks like all the work is done, but there is one more thing left to do. If you try to create a new database you'll notice that the files will not land in you `C:\data` (inside container) folder. Try simple SQL query:

```sql
CREATE DATABASE TestDb;

SELECT
  name,
  physical_name
FROM sys.master_files
WHERE name='TestDb'
```
Not the result we really expected:
{% include center-img.html images="articles/2022/sql-image-sitecore/Code_S1lvaHqUTV.png" %}
