# Simcoe County Deployment Guide

This guide will help you setup the Viewer, API and all its supporting websites on your own machine/server.

## **GeoServer and PostGres/PostGIS setup.**

I'm not going into too much detail here but I'll give you a little direction on our setup.

We're currently running GeoServer **2.16.1** and PostGres **12.1**

### **GeoServer**

- Download the latest GeoServer from http://geoserver.org/download/
- Select the Platform Independant library.
- Follow the ReadMe inside the zip to install
- Layer Groups should have the "Named Tree" option selected for the Mode.
- It's recommended change your data_dir location. We have it set to a network UNC Path.

### **PostGres/PostGIS**

- Download the latest PostGres from https://www.postgresql.org/download/windows/
- Select the installer link "certified by EnterpriseDB" at the top of the page.
- After the installation is complete, use the "Stack Builder" program to install PostGIS and any other extensions you want. More Info found here: https://postgis.net/install/

## **Node JS**

Simply install Node on your server from https://nodejs.org/en/download/
We're running **10.16.3**

## **PM2**

PM2 is not required but highly recommended. This will manage all your node apps once in production.
This install can be tricky. There are many guides out there, but here's what worked for me.

Edit: If the method below does not work for you, a fellow user found this link useful.
https://stackoverflow.com/questions/38185590/pm2-command-not-found

- We have pm2 running as a designated admin user on our server. Its a domain user but I don't think that matters.
- Login to your server with that Admin user
- Install pm2 on the cmd with "npm install pm2"
- Create and Environment Variable called "PM2_HOME" and set it to your admin users directory. "C:\Users\yourAdminUsername\AppData\Roaming\npm"
- Logoff and login with a different account. Type this in a cmd to see if things are working "pm2 ls". As long as you don't get errors you're good.
- Create a batch file that will start PM2. It's simply "pm2 resurect" to start it up.
- Create a scheduled task to run that bat file when the server boots.
- We have a local folder that contains all our websites and proxy. This keeps things clean and organized.
- Offical website is here: https://pm2.keymetrics.io/

## **Reverse Proxy**

All our web traffic will run through this and be directed to the proper ports.

- Copy the proxy project to a folder on your server. https://github.com/county-of-simcoe-gis/SimcoeCountyProxy
- In the proxy project, edit the config.json
  - If you didn't change your default GeoServer port, the existing entry should work for you. This will direct all incoming traffic to http://YourServer/geoserver to your GeoServer. This is what allows us to resolve https://opengis.simcoe.ca/geoserver
    "route": "/geoserver",
    "address": "http://localhost:8080/geoserver"
- Add the proxy to pm2. In a cmd window, "pm2 start "C:\YourProxyFolder\app.js" --name proxy"
- Save this entry with "pm2 save"
- You should now be able to resolve to your GeoServer with a friendly host header or the name of your server.
- With the scheduled task we created in the PM2 section above, we should now be able to reboot our server and pm2 should start up our proxy.
- After reboot in a cmd, type "pm2 ls". It should list your proxy.

## **SQL PostGres script**

- I've consolidated all our sql scripts into one file found here: https://github.com/county-of-simcoe-gis/SimcoeCountyWebApi/blob/master/sqlScript.sql
- Ensure you run this on one of your databases (No PostGIS required) and we'll point to it in the WebApi below.

## **Web Api**

This the main api that is core to this project.

- We have 2 databases.
  - tabular - This is a non-spatial database that we use for AppStats, MyMaps, Search, etc.
  - weblive - This is our PostGIS database that contains all our layers. The only thing the Web API uses this database for is to run GeoJSon/PostGIS commands (e.g. Buffer). **Therefore, this can be an empty PostGIS database.**
- Edit the config.json

  - Point to your databases (tabular and weblive)
  - Change the email details to your own organization
  - Change the feedback Url to wherever you have it hosted. Project is here: https://github.com/county-of-simcoe-gis/SimcoeCountyFeedback
  - We use Google Captcha to protect the Feedback page. This is optional, but you'll need to comment it out of the Feedback app if you don't want it.
  - Map Url is used to link in the feedback email
  - Allowed Origins is an extra layer of safety to ensure that originating calls come from your own server.

- Copy the WebApi project to a folder on your server. https://github.com/county-of-simcoe-gis/SimcoeCountyWebApi
- If you're not copying this from an already built project, you'll need to get the node_modules by running "npm install"
- Add it to pm2. "pm2 start "C:\YourWebApiFolder\app.js" --name api"
- Save it in pm2. "pm2 save"
- This, along with the proxy will now auto start on reboot.

## **Setup All Other Websites on PM2**

All other websites simply need a container to run with minimal configuration. **Remember to change the ports accordingly in your proxy config.json**

Check for config.json in the root of the src folders of each website to point to your server. The main SimcoeCountyWebViewer for example, can be modified to include/exclude the Tools/Themes.

I'll use SimcoeCountyWebViewer as an example here but you'll do the same for SimcoeCountyFeedback, SimcoeCountyAppStats, SimcoeCountyFeatureReports and SimcoeCountyGeoServerLayerInfo.

- Copy our sample hosting project to your server. We will also use this as a template to run all our other websites. "https://github.com/county-of-simcoe-gis/SimcoeCountySampleNodeHostingContainer
- Rename the folder to SimcoeCountyWebViewer.
- After you've ran "npm run build" on the website locally, copy the build output folder to the build folder of the Hosting container we created above.
  **Each website will have it's folder container**
- In my devloper section below I describe how I make this part easier
- Add it to pm2. In a cmd window, "pm2 start "C:\YourSimcoeCountyWebViewerFolder\app.js" --name viewer"
- Save this entry with "pm2 save"

## **Search**

The search processor runs in Python. We're running 3.7 but it may work in 2.7.
**GeoServer WFS needs to be enabled on the layers to be searched.**

- Insert records into tbl_search_layers that match your data. Each record represents a layer you want to be searchable. Sample insert statements can be found here at the bottom of the script: https://github.com/county-of-simcoe-gis/SimcoeCountyWebApi/blob/master/sqlScript.sql
- Install Python 3.7
- Install packages. In cmd, "pip install psycopg2" and "pip install postgres"
- Copy the project to your server: https://github.com/county-of-simcoe-gis/SimcoeCountySearchProcessor
- Test the script by running (c:\python37\python.exe "C:\SearchProcessorFolder\main.py")
- At this point, if you have the WebApi configured, your layer can now be searched in the main viewer.
- I've got it setup to run nightly with Task Scheduler. Create a new task to run at the time you want (e.g. 2:00 A.M.) and create a bat file that calls python. c:\python37\python.exe "C:\SearchProcessorFolder\main.py"

## **Developer Setup/Considerations**

So the typical debugging scenario is having the SimcoeCountyWebViewer and SimcoeCountyWebAPI running locally. Here's my setup.

- I'm running VS Code for python and web development (https://code.visualstudio.com/download)
- Side note: For the express apps (Api), I've got the nodemon package installed, this allows you to make edits and when you save, it automatically restarts your local server. Really Handy.
- Get the SimcoeCountyWebApi running locally. By default this is port 8085. ("nodemon npm start" to start it up )
- Once you have the api running, get the viewer running and edit the config and change the "apiUrl" entry to http://localhost:8085.
- You can now debug the api and the viewer.
- Once you're ready to deploy your code to the server, change your api url back to production in viewer config.
- run npm run build
- For copying, I use a bat file. Create a bat file at the root of each of your projects. I use robocopy.

If you need to push a **React Website** (e.g. SimcoeCountyWebViewer), first run "npm run build" then run the bat file. (robocopy D:\PathToYourViewer\build \\YourServerName\c\$\YourViewerNodeContainer\build /e /xo /FFT /XA:H /R:1 /W:1 /V /TEE)

If you need to push the **WebApi** (e.g. SimcoeCountyWebApi), run the bat file. (robocopy D:\PathToYourWebApi \\YourServerName\c\$\YourWebApiFolder /e /xo /FFT /XA:H /R:1 /W:1 /V /TEE)
**When copying the web api, you need to restart it. Run pm2 restart IdOfYourWebApi on your server after copying new files** Run "pm2 ls" to see what the id is for your web api.

This concept works for any of the projects. Change the config to your localhost:port and you're good to go.

## **How do you like this guide?**

This guide is for you, so please provide feedback if you think a section deserves more attention/clarification.

**Please submit this feedback as an issue at: https://github.com/county-of-simcoe-gis/SimcoeCountyDeploymentGuide/issues**
