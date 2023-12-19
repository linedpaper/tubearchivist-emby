## Tube Archivist emby Integration
Import your Tube Archivist media folder into emby

![home screenshot](assets/screenshot-home.png?raw=true "emby Home")

This repo looks for regular contributors. If this is a useful integration, consider improving upon it.   

This requires Tube Archivist v0.4.4 or later for API compatibility.

and request emby rest api refrence:
https://betadev.emby.media/reference/RestAPI.html

edited based on tubearchivist-jf https://github.com/CommanderRedYT/tubearchivist-jf
## Core functionality
- Import each YouTube channel as a TV Show
- Each year will become a Season of that Show
- Load artwork and additional metadata into emby

## How does that work?
At the core, this links the two APIs together: This first queries the emby API for YouTube videos for any videos that don't have metadata to then populate the required fields from Tube Archivist. Then as a secondary step this will transfer the artwork.

This doesn't depend on any additional emby plugins, that is a stand alone solution.

This is a *one way* sync, syncing metadata from TA to emby. This syncs in particular:
- Video title   
- Video description
- Video date published      
- Channel name
- Channel description 

## Setup emby

Take a look at the example docker-compose.yml provided.

0. Add the Tube Archivist **/youtube** folder as a media folder for emby.
    - IMPORTANT: This needs to be mounted as **read only** aka `ro`, otherwise this will mess up Tube Archivist.  

1. Add a new media library to your emby server for your Tube Archivist videos, required options:
    - Content type: `Shows`
    - Displayname: `YouTube`
    - Folder: Root folder for Tube Archivist videos
    - Deactivate all Metadata downloaders
    - Automatically refresh metadata from the internet: `Never`
    - Deactivate all Image fetchers

2. Let emby complete the library scan
    - This works best if emby has found all media files and Tube Archivist isn't currently downloading.
    - At first, this will add all channels as a Show with a single Season Unknown.
    - Then this script will populate the metadata.

3. Backdrops
    - In your emby installation under > *Settings* > *Display* > enable *Backdrops* for best channel art viewing experience.

## Install with Docker
An example configuration is provided in the docker-compose.yml file. Configure these environment variables:
  - `TA_URL`: Full URL where Tube Archivist is reachable
  - `TA_TOKEN`: Tube Archivist API token, accessible from the settings page
  - `JF_URL`: Full URL where emby is reachable
  - `JF_TOKEN`: emby API token
  - `JF_FOLDER`: Folder override if your folder is not named "YouTube" on your Filesystem.
  - `LISTEN_PORT`: Optionally change the port where the integration is listening for messages. Defaults to `8001`. If you change this, make sure to also change the json link for auto trigger as described below.

Mount the `/youtube` folder from Tube Archivist also in this container at `/youtube` to give this integration access to the media archive.

### Manual trigger
For an initial import or for other irregular occasions, trigger the library scan from outside the container, e.g.:
```bash
docker exec -it tubearchivist-emby python main.py
```

### Auto trigger
Use the notification functionality of Tube Archivist to automatically trigger a library scan whenever the download task completes in Tube Archivist. For the `Start download` schedule on your settings page add a json Apprise link to send a push notification to the `tubearchivist-emby` container on task completion, make sure to specify the port, e.g.:

```
json://tubearchivist-emby:8001
```


## Install Standalone
1. Install required libraries for your environment, e.g.
```bash
pip install requests
```
2. rename/copy *config.sample.json* to *config.json*.
3. configure these keys:
	- `ta_video_path`: Absolute path of your /youtube folder from Tube Archivist
	- `ta_url`: Full URL where Tube Archivist is reachable
	- `ta_token`: Tube Archivist API token, accessible from the settings page
	- `jf_url`: Full URL where emby is reachable
	- `jf_token`: emby API token
    - `jf_folder`: Name of the folder where TubeArchivist puts the files into

Then run the script from the main folder with python, e.g.
```python
python app/main.py
```

## Limitations
You can only have *one* folder called **YouTube** in your emby.

emby needs to be able to see the temporary season folders created by this extensions. You will see messages like `waiting for seasons to be created` before you will run into a `TimeoutError`, if that doesn't happen in a reasonable time frame.

Some ideas for why that is:
- Your JF busy, too slow or is already refreshing another library and is not picking up the folder in time.
- JF doesn't have the permissions to see the folder created by the extension.
- You didn't mount the volumes as expected and JF is looking in the wrong place.

## Migration problems
Due to the filesystem change between Tube Archivist v0.3.6 to v0.4.0, this will reset your YouTube videos in emby and will add them as new again. Unfortunately there is no migration path.

To import an existing Tube Archivist archive created with v0.3.4 or before, there are a few manual steps needed. These issues are fixed with videos and channels indexed with v0.3.5 and later.

Apply these fixes *before* importing the archive.

**Permissions**  
Fix folder permissions not owned by the correct user. Navigate to the `ta_video_path` and run:

```bash
sudo chown -R $UID:$GID .
```
