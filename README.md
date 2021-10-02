


# Reshenie Dashboard
## _project overview by Sunwook_

[![N|Solid](https://cldup.com/dTxpPi9lDf.thumb.png)](https://nodesource.com/products/nsolid)
[![Build Status](https://travis-ci.org/joemccann/dillinger.svg?branch=master)](https://travis-ci.org/joemccann/dillinger)

Reshenie's dashboard project consists of multiple projects combined into a web-app to view information offered by Reshenie.

- [Reshenie Dashboard](#reshenie-dashboard)
  * [_project overview by Sunwook_](#-project-overview-by-sunwook-)
  * [Features](#features)
  * [Technologies](#technologies)
  * [Installation](#installation)
  * [Implementation](#implementation)
    + [Battery/Connectivity](#battery-connectivity)
      - [Main function](#main-function)
      - [getBatteryPercentage()](#getbatterypercentage--)
      - [connectDB()](#connectdb--)
      - [getBatteryStatus()](#getbatterystatus--)
      - [automation](#automation)
    + [X/Y/ZRMS data](#x-y-zrms-data)
      - [Main function](#main-function-1)
    + [combineJson](#combinejson)
      - [merge_JsonFiles()](#merge-jsonfiles--)
      - [FTP connection](#ftp-connection)

## Features
- View and change list of all clients and its factories
- View list of sensors in selected factory and its data (connectivity, battery usage, MAC ID, sensor name, sensor image ..etc)
- 2 Plot.ly graphs to chart chosen information within the selected data range
- cal-heatmap.com to select data range
- 2 tables to select time range within chosen date 

## Technologies
Reshenie's dashboard uses a number of open source projects to work properly:

- [Cal-Heatmap](https://cal-heatmap.com/) - javascript module to create calendar heatmap to visualize time series data
- [jQuery] - duh (A fast, small, and feature-rich JavaScript library)
- [D3.js](http://d3js.org/) - A JavaScript library for manipulating documents based on data.
- [PHP](http://php.net/) - Popular general-purpose scripting language that is especially suited to web development.
- [JavaScript](https://en.wikipedia.org/wiki/JavaScript) - Programming language of HTML and the web.
- [Python](https://www.python.org/) - Programming language that lets you work quickly and integrate systems more effectively.
- [SQL](https://en.wikipedia.org/wiki/SQL) - Stands for structured query language used with relational databases.
- [MySQL](https://www.mysql.com/) - One of the world's most popular open source databases.
- [Less](http://lesscss.org/) - As an extension to CSS that is also backward compatible with CSS. This makes learning Less a breeze, and if in doubt, lets you fall back to vanilla CSS.

## Installation
Currently the main webpage reshenie.co.kr is hosted on [cafe24](cafe24.com) with the configuration of
- UTF-8
- PHP7.3
- MariaDB-10.0.x

Install the dependencies and start the server

Out of the available 700MB of storage on the web server the website in its current state (27/9/2021) is using 334/700MB, meaning more storage will be needed to fully integrate and complete the project.

## Backend Implementation
This section will break down each features/program used within the project into its purpose and usage.

### Battery/Connectivity
#### Main function
Within the main function list the sensor's MAC ID and its corresponding sensor name in list *macIds*, *macIdNames* and update the server node's IP in the first parameter for function *getBatteryStatus()*
```python
macIds = ['38:91:12:21', 'e0:10:9a:cd', 'ec:1a:69:5f','fe:1e:09:59']
macIdNames = ['Kowon_Vib_BackMotor', 'Kowon_Vib_BackCrank', 'Kowon_Vib_FrontMotor','Kowon_Vib_FrontCrank']
for id, names in zip(macIds,macIdNames):
    print(id)
    (values, dates) = getBatteryStatus("25.100.199.132", id)
    connectionStatus, batteryPercentage = getBatteryPercentage(values)
    print(batteryPercentage)
    connectDB(connectionStatus, batteryPercentage, names)
```
#### getBatteryPercentage()
The function *getBatteryPercentage()* servers 2 purposes
- map the battery voltage to a percentage scale
- if no battery voltage for the sensor is detected return its *connectionStatus* as disconnected and *batteryPercentage* as 0
```python
def getBatteryPercentage(values):
    m = interp1d([2.1,3.24],[1,100])
    # print(values)
    if not values:
        print("NO battery or/and NO connection")
        batteryPercentage = 0
        connectionStatus = "disconnected"
    else:
        print("battery and connection GOOD")
        connectionStatus = "connected"
        batteryPercentage = m(values[-1])

    return connectionStatus, batteryPercentage
```
#### connectDB()
The function *connectDB()* makes a connection to the webapp's database through a mysql connector and updates the data for sensors entered in the main function
```python
def connectDB(connectionStatus, batteryPercentage, sensorName):
    mydb = mysql.connector.connect(
        host="*enter host*",
        user="*enter user*",
        password="*enter password*",
        database="*enter database name*",
        port="3306"
    )
    mycursor = mydb.cursor()
    sql = "UPDATE gorus.sensors SET connectionStatus=%s, batteryPercentage=%s WHERE sensorName=%s;"
    val = (connectionStatus, str(batteryPercentage), sensorName)
    mycursor.execute(sql, val)
    mydb.commit()
    print(mycursor.rowcount, "record(s) affected")
```
#### getBatteryStatus()
Function *getBatteryStatus* returns the voltage and its date data from IQunet's database
```python
def getBatteryStatus(serverIP, macId):
    logging.basicConfig(level=logging.INFO)
    logging.getLogger("opcua").setLevel(logging.WARNING)
    serverUrl = urlparse('opc.tcp://{:s}:4840'.format(serverIP))
    endTime = datetime.datetime.now()
    startTime = endTime - datetime.timedelta(hours=10)

    print('start time ', startTime)
    print('end time ', endTime)

    browseName=["accelerationPack","axis","batteryVoltage","boardTemperature",
                "firmware","formatRange","gKurtX","gKurtY","gKurtZ","gRmsX","gRmsY",
                "gRmsZ","hardware","mmsKurtX","mmsKurtY","mmsKurtZ",
                "mmsRmsX","mmsRmsY","mmsRmsZ","numSamples","sampleRate"]

    (values,dates) = DataAcquisition.get_sensor_data(
        serverUrl=serverUrl,
        macId=macId,
        browseName=browseName[2],
        starttime=startTime,
        endtime=endTime
        )
    return values, dates
```
#### automation
To setup this feature to run automatically use the schedule function and loop it self within an infinite while loop with a sleep time value
```python
schedule.every(1).second.do(*enter function*)

while True:
    schedule.run_pending()
    time.sleep(1)

```
### X/Y/ZRMS data
#### Main function
Fill out sensor's server IP, MAC ID and the correct axis from list *browseName* into correct variables.
Currently the program is set to acquire all data from 365 days ago, this was for catelogging purposes and there is no need to grab acquired data again.
```python
rmsList = ["gRmsX", "gRmsY", "gRmsZ"]
...
serverIP = "x.x.x.x"
...
macId = 'x:x:x:x'
...
endtime = datetime.datetime.now()
starttime = endtime - datetime.timedelta(days=365)

browseName = ["accelerationPack", "axis", "batteryVoltage", "boardTemperature",
              "firmware", "formatRange", "gKurtX", "gKurtY", "gKurtZ", "gRmsX", "gRmsY",
              "gRmsZ", "hardware", "mmsKurtX", "mmsKurtY", "mmsKurtZ",
              "mmsRmsX", "mmsRmsY", "mmsRmsZ", "numSamples", "sampleRate"]

(values, dates) = DataAcquisition.get_sensor_data(
    serverUrl=serverUrl,
    macId=macId,
    browseName=browseName[9],
    starttime=starttime,
    endtime=endtime
```
Saves acquired data into entered path to be processed in *combainedJson* program
```python
values = []
epochDates = []
for date in dates:
    date_time = datetime.datetime.strptime(date, "%Y-%m-%d %H:%M:%S")
    date_time_seconds = time.mktime(date_time.timetuple())
    values.append(1)
    epochDates.append(date_time_seconds)

data = [{"epochDates": d, "value": v} for d, v in zip(epochDates, values)]

with open(' *enter path to file "X_Kowon_Vib_BackCrank_epoch_data.json" ', 'w') as json_file:
    json.dump(data, json_file, indent=4)
```

### combineJson
The combineJson program combines all the JSON files in the directories listed in the list *files*
```python
files = [
' *path to X_Kowon_Vib_BackCrank_epoch_data.json* ',
' *path to Y_Kowon_Vib_BackCrank_epoch_data.json* ',
' *path to Z_Kowon_Vib_BackCrank_epoch_data.json* ']
```
#### merge_JsonFiles()
```python
def merge_JsonFiles(filename):
    result = list()
    for f1 in filename:
        with open(f1, 'r') as infile:
            result.extend(json.load(infile))
    with open(' *path to XYZ_Kowon_Vib_BackCrank_epoch_data.json* ', 'w') as output_file:
        json.dump(result, output_file)
```
#### FTP connection
After X/Y/Z axis json files are combined they are uploaded to the webserver through FTP
```python
merge_JsonFiles(files)

HOSTNAME = " *enter hostname* "
USERNAME = " *enter username* "
PASSWORD = " *enter password* "

import ftplib
import os
filename = "XYZ_Kowon_Vib_BackCrank_epoch_data.json"
ftp = ftplib.FTP(HOSTNAME)
ftp.login(USERNAME, PASSWORD)
ftp.cwd("/www/dashboard/data")
path = (r' *path where file in *filename* is located* ')
path_encode = path.encode('UTF-8')
os.chdir(path_encode)
myfile = open(filename, 'rb')
ftp.storbinary('STOR ' + filename, myfile)
myfile.close()
```
## Workspace setup
#### FTP
As a personal preference I used FileZilla for a FTP client
#### SQL Client
Any SQL client tool/database management software will do the job but I used DBeaver as a personal preference. Other options include MySQL Workbench - a more mainstream option
#### local testing
XAMPP is a free and open-source cross-platform web server solution stack package developed by Apache Friends, consisting mainly of the Apache HTTP Server, MariaDB database, and interpreters for scripts written in the PHP and Perl programming languages.
#### Development platform
As my choice of code editor I've used Visual Studio Code by Microsoft. VSCode was used to debug, syntax highlight, intelligent code completion, snippets, code refactoring, and Git usage.

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [dill]: <https://github.com/joemccann/dillinger>
   [git-repo-url]: <https://github.com/joemccann/dillinger.git>
   [john gruber]: <http://daringfireball.net>
   [df1]: <http://daringfireball.net/projects/markdown/>
   [markdown-it]: <https://github.com/markdown-it/markdown-it>
   [Ace Editor]: <http://ace.ajax.org>
   [node.js]: <http://nodejs.org>
   [Twitter Bootstrap]: <http://twitter.github.com/bootstrap/>
   [jQuery]: <http://jquery.com>
   [@tjholowaychuk]: <http://twitter.com/tjholowaychuk>
   [express]: <http://expressjs.com>
   [AngularJS]: <http://angularjs.org>
   [Gulp]: <http://gulpjs.com>

   [PlDb]: <https://github.com/joemccann/dillinger/tree/master/plugins/dropbox/README.md>
   [PlGh]: <https://github.com/joemccann/dillinger/tree/master/plugins/github/README.md>
   [PlGd]: <https://github.com/joemccann/dillinger/tree/master/plugins/googledrive/README.md>
   [PlOd]: <https://github.com/joemccann/dillinger/tree/master/plugins/onedrive/README.md>
   [PlMe]: <https://github.com/joemccann/dillinger/tree/master/plugins/medium/README.md>
   [PlGa]: <https://github.com/RahulHP/dillinger/blob/master/plugins/googleanalytics/README.md>
