Welcome to the AirSense-Network. AirSense is simply a esp32 node that can caputure various data related to the air quality. This repo contains all the code you need to setup a network of AirSense nodes and send the data to a central mqtt broker and store it in a database, here i am using influx db as the database and mosquitto for the mqtt broker.

# Setting up MQTT Broker

*Why mqtt? Its a more reliable and light weight protocol that can be deployed in areas with unstable networks and the reliability can be controlled, by modifying the quality of service(qos) parameter. You can read and learn more about mqtt here: [mqtt-academy](https://cedalo.com/mqtt-academy/)*

I will be using mosquitto here as my mqtt broker as its open source, lightweight and easy to configure and scalable.

#### Installing mosquitto 

Refer to [mosquitto docs](https://mosquitto.org/download/) for detailed installation guide for your os.

**In linux machines you might have to enable and start the mosquitto service after installing**

To verify if your installation was successful, open a terminal and run
`mosquitto_sub -h localhost -t test`

This is will create a mqtt subscriber that listent to the test topic on the mosquitto broker running in localhost

On a new terminal run
`mosquitto_pub -h localhost -t test -m "hello, world"`
this will send the message hello world to the mqtt broker and will be desplayed in the terminal that was subscribed to the topic test.

If that's working, you are good to go and can exit from both terminals. Else go the mosquitto docs for troubleshooting.

#### Disabling mosquitto authentication

Now before proceeding we have to change a small mosquitto configuration to allow esps to connect to the broker without any authentication. Ideally you should have an authenticaion which I will be implementing soon, but for now disabling authentication helps to setup everything and play around wtih the network more easily.

Find the mosquitto.conf file. For linux systems you can find it here
`/etc/mosquitto/mosquitto.conf`

Add these two lines to the file in below
```
listener 1883
allow_anonymous
```
Save this change and restart your mosquitto service.

 >*If you are having issues with mosquitto_pub and mosquitto_sub step maybe adding listner 1883 to the mosquitto.conf file can help, or use it to setup a different port if you need.* 

# Setting up the Esp firmware.

We will be using **Esp home** for a low code and easily configurable setup.
 
 #### Installing ESP Home 
 ***⚠️I would highly recommend to run the below instructions in a python virtual environment*** 
 Installing esp home just by installing it using pip. 
 `pip3 install esphome.`

#### Writing esp firmware using esp home

I have provided a folder named `esphome` with the config files I used, but **i had to write config files for each esp to set their unique id and stuff, which i think can automated using esphome config settings, have to figure that out later**. For now you can use the config files i have provided. I am using `seeed_studio_xiao_c3` boards. you might have to write your config for your board which pretty straightforward. Follow the steps below to write your own custom config.

Run
`esphome wizard <config_file_name.yaml>`
You will be asked some prompts to set the important settings like name, board, wifi setup, ota update atuthentication setup. Answer it according your setup and you are good to go.

#### Setting up mqtt client communication

Now it's time to setup the mqtt client. 

just copy and paste the below lines to the config file you just created.

```
mqtt:
	broker: <mqtt broker ip adress>
	port: 1883  # 1883 is the default mqtt port, change if you are needed
	birth_message:
		topic: "test"
		payload: "C3-1 is online"	# meassage to send when the esp is alive

```

This code will connect to the mqtt broker and sends the message to the test topic.

Now to setup the sensor code. Right now I'm just sending internal temperature of the esp to the database. I will be adding more sensors to the node as the point of this project is to actually get all the data about the air quality.
Add below code to the config as well.

You can refer to the esphome documentation to understand this code better.  Taking a look at lambda function might be really helpful as it tend to be a bit confusing.

[internal temperature sensor documentation](https://esphome.io/components/sensor/internal_temperature.html)
[esphome base sensor docs](https://esphome.io/components/sensor/#config-sensor)
```
sensor:
 13   - platform: internal_temperature
 12     name: "ESP32 Internal Temperature"
 11     id: internal_temp_sensor
 10     update_interval: 30s
  9     on_value:
  8       then:
  7         - logger.log:
  6             format: "Internal Temp: %.1f"
  5             args: ["id(internal_temp_sensor).state"]
  4         - mqtt.publish:
  3             topic: "test"
  2             payload: !lambda |-
  1               return "c3-1 temp: " + std::to_string(id(internal_temp_sensor).state);
  ```
***TODO: Udpate the formatting***

Now every 30 secs esp wil be sending the data to the mqtt broker. 

#### Flashing the firmware to the esp

For the first time, you need to have the esp connected your devices using usb, after that you can send updates over the air(ota) if the esp is powered and connected to the same network.

To flash the firmware run
`esphome run <config_file_name.yaml> --device <port esp is connected to>`

This might take a minute to execute, once its done you will see the esp seraching for all the available networks and connecting to the network you configured and then all the logs including the data that is being send. Once that's verified you can exit the process by hitting Ctrl+C

***You can create a mostquitto_sub that subscribes to the test topic to see if its working properly. Just make sure everything is on the same server.***

# Setting up Influxdb


>I think before i proceed i think i should explain about influxdb and some guidelines or warnings. Influxdb is a db specialized for storing timeseris data. As we were collecting datapoints from different timestamps influxdb becomes an obvious choice. But influxdb has gone through a roller cost of updates with major changes across versions and being not back compatibilites. I hope they stick to current tooling from moving forward. But depending on versions are the tooling changes a bit and there is a learning curve between versions. I am going with the latest version and some toolings(influxdb explorer) are in beta but for now it works. Below I will provide instructions for my setup, if you don't want' to follow it feel free to play around.

Refer to the [influxdb docs](https://docs.influxdata.com/influxdb3/core/install/) for detailed installation and setup. 

I am also using the following db setup when i start the db instance
`influxdb3 serve --node-id test00 --object-store file --data-dir ~/.testinfluxdb3`

I am using a file based storage for testing and to have data persistance since i constanntly exit the process and easily test stuff around. Might change the configuration a bit more suitatble for our task and my learning gets more solid.

#### Setting up authentication.

Refer to the [influx db  getting started](https://docs.influxdata.com/influxdb3/core/get-started/) to setup the authentication and please keep the generated tokens safe. Don't make it public unless you want everyone to access your database and mess it up.

#### Setting up influxdb explorer

> Influxdb explorer is gui to work with you database. Its still in public beta but works fairly fine with minute lags and here and there, for now its get the job done and makes life way more easier.

**You will be needing docker for the following step, so make sure you have installed docker and running before proceeding**

```
cd influxdb
mkdir -m 700 ./db
mkdir -m 755 ./config
mkdir -m 755 ./ssl

docker run --detach \
--name influxdb3-explorer \
--publish 8888:80 \
--publish 8889:8888 \
--volume $(pwd)/config:/app-root/config:ro \
--volume $(pwd)/db:/db:rw \
--volume $(pwd)/ssl:/etc/nginx/ssl:ro \
quay.io/influxdb/influxdb3-explorer:latest \
--mode=admin
```
This will spawn a container running influxdb explorer which you can access from http://localhost:8888.

> if you are setting up everything in a virtual machine without gui, you can access the explorer in your host machine by going to the url http://<your_vm_ip>:8888/

From the gui you can connect to the database by clicking on Add server and entering your database url, usually http://localhost:8181

Go to the authentication tab and authenticate using your influxdb3 token you generated when setting up influxdb to perform various actions through the gui.

# Setting up telegraf

Now we have the mqtt client, broker, and database setup. The only thing left is how to send the data from the mqtt broker to the database. This is where telegraf comes. Telegraf is a open source tool developed by influx to ingest various types of data, process or aggregrate and output in different manners in a low code manner. 

#### Installing telegraf.

I'm getting tired, please refer to the [telegraf docs](https://docs.influxdata.com/telegraf/v1/install/) for installation.

Once the installation is done and verified you can naviagte to `/telegraf/telegraf.conf` to use the telegraf config i am using. Make sure to update the influxdb authentication to your generated token in the telegraf.conf file.

Test telegraf setup using
`telegraf --config telegraf.conf --test`

Now the data shoudl be aggregated from mosquitto and sent to the database in a table called internal_temps. You can now query, visualize the data from influxdb explorer.

That's  it. Best of Luck!!








