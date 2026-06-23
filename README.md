# Roundabout Traffic Analysis

Production version with multiple cameras, an InfluxDB time-series database, and Grafana dashboards.

This program analyzes incoming traffic at a roundabout section. The algorithm determines the congestion level of the adjacent roads and outputs interactive statistics.


## Installation and launch

### Clone the repository

```
git clone https://github.com/galelisette/traffanalys
```

### Configure environment variables

After that, you need to create a file with environment variables in the project's root directory, which will be passed into the Grafana and Influx containers. To do this, create a `.env` file and specify text like the following with the passwords and logins for the services:

```
INFLUXDB_ADMIN_USER=admin
INFLUXDB_ADMIN_PASSWORD=admin
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin
KAFKA_USERNAME=traffic
KAFKA_PASSWORD=traffic-secret
```

### Run the project

Next, launch the project using this command:

```
docker compose -p traffic_analyzer up -d --build
```

### Adding cameras

Each new camera is added in the compose file as one additional backend instance, `traffic_analyzer_camera_{n}`, where you only need to specify a different source (`src`) and configuration via the service's environment variables.

## Running locally in Python without additional microservices

```
# install the libraries:
python -m pip install --upgrade pip
pip install "numpy<2"
pip install cython_bbox==0.1.5 lap==0.4.0 
pip install torch==2.3.1 torchvision==0.18.1 --index-url https://download.pytorch.org/whl/cu121
pip install -r requirements.txt

# run the code:
python main_optimized.py pipeline.send_info_kafka=False
```


## Project architecture

The project is a real-time video analysis system that works with RTSP streams or MP4 files. The main service, `traffic_analyzer_camera_{n}`, processes frames, extracts analytical data (such as the number of vehicles in the roundabout area and the congestion of adjacent roads), and sends it to the Kafka message broker. For each camera, real-time statistics are recorded in its own topic, `statistics_{n}`. Data from Kafka is automatically written to the InfluxDB time-series database using Telegraf. InfluxDB is optimal for storing streaming data thanks to its high performance and support for large volumes of information.

Grafana is used to visualize the data: it connects to InfluxDB and displays the analytics as interactive dashboards. This makes it possible to track key metrics in real time, build charts, and analyze trends.

### Main components

1. **traffic_analyzer_camera_{n}** — processes the video stream numbered `n` and sends data to Kafka.
2. **Kafka** — temporary storage and transfer of data.
3. **Telegraf** — transfers data from Kafka to InfluxDB.
4. **InfluxDB** — stores the analytical data.
5. **Grafana** — visualizes data from InfluxDB in interactive dashboards.
6. **Nginx** — acts as a reverse proxy that combines all the resulting Flask streams of processed video on a single port with different endpoints. This makes it convenient to manage access to the video streams and provides a single entry point for all cameras.

![Project architecture](https://github.com/galelisette/traffanalys/blob/main/archit.png)

## How the main video processing service works

Let's look at how the code of the main video stream processing service is implemented.

Each frame (a `FrameElement` object) passes sequentially through the nodes, and more and more information is gradually added to the object's attributes.

![How the main video processing service works](https://github.com/galelisette/traffanalys/blob/main/video_processing_pipeline.png)

## Usage

Before launching, you need to specify all the desired parameters in the `configs/app_config.yaml` file. After that, you can run the code.

To run the project with a specific video, you need to specify the path to it in the docker-compose environment variable. Instead of a file path, you can specify a link to an RTSP stream. In the same container environment variables, you can also specify the path to a JSON file containing the coordinates of the polygons for the adjacent roads.

### Launch options for MP4 files

`main.py` — the project's main code, which runs frames through all the nodes in a loop.

`main_optimized.py` — an optimized version of `main.py` using multiprocessing. It achieves higher processing speed (over 35 frames per second), since all resource-intensive operations are distributed across independent processes running in parallel.

### Additional launch options (real-time RTSP streams only)

`main_stream_optimized.py` — a version for working with real-time streaming video that processes only the most up-to-date frames without using a buffer. This is achieved by processing frames in a separate process, while the main process always takes only the latest available frame for processing.

`main_stream_optimized_v2.py` — an improved version of `main_stream_optimized.py`. The main difference is that if one of the processes terminates or fails, the other process is automatically terminated as well. Process state is monitored via the `process.is_alive()` method, which provides more reliable control over the process lifecycle.

### Examples

Example of the algorithm's output with statistics: each car is displayed in a color corresponding to the road it arrived from at the roundabout, plus the number of visible cars is shown, plus the values of the incoming flow intensity (number of cars per minute from each incoming road).

This is displayed when `show_node.show_info_statistics=True` is set in the configuration.

![Example1](https://github.com/galelisette/traffanalys/blob/main/cam1.gif)

You can disable the statistics window by setting `show_node.show_info_statistics=False` in the configuration.

To see the processing FPS as in the first example shown, set `show_node.draw_fps_info=True` in the config.

Example of the car-tracking results display mode (each ID is shown in its own unique color).

![Example2](https://github.com/galelisette/traffanalys/blob/main/cam2.gif)

This is displayed when `show_node.show_track_id_different_colors=True` is set in the configuration.

