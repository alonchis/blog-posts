Hi, it's my first time writing one of these guides. I spend so much time lurking on the internet for leisure and researching interesting stuff. I’m a bit of a nerd,  so most of that is spend on tech projects. I’m by no means a professional, just a very curious guy. The main motivation for writing this guide is, as Adam Savage once put it, [“the difference between screwin’ around and science… is writing it down”.](https://www.youtube.com/watch?v=BSUMBBFjxrY)

This is a guide to assemble a raspberry pi weather station.  It will measure temperature, humidity, and (once I get my hands on a sensor) atmospheric pressure. This can be used as a data point for openweathermap.org too. I would like to figure out in the future if this can also be used as a temperature sensor to feed ~~Nest~~ Google's Nest, TBD.

##Materials:
I don’t receive any commission/affiliate bonuses etc from these link. They’re just for reference.
* [Raspberry Pi 3B+](https://www.adafruit.com/product/3775) (though any Pi, even a Zero W will work), and its necessary components such as power cable, ethernet/wifi, SD card. If you don’t know how to install the OS, [here is a good guide for that.](https://www.adafruit.com/product/3775)
* [AM2302](https://www.adafruit.com/product/393) (aka DHT22): This is a temperature and humidity sensor. It's the DHT22 sensor in a bigger enclosure and includes the necessary resistor inside. A DHT22 will work too, you’d just need to add a 4.7Kohm resistor.
* [DS18B20](https://www.adafruit.com/product/381): Waterproof sensor.
* Some sort of weatherproof housing: (Still in discovery for a good solution, as of now, [this is a potential lead](https://www.amazon.com/LeMotech-Dustproof-Waterproof-Electrical-150mmx110mmx70mm/dp/B075DHRJHZ/ref=sr_1_7?crid=ETZJDA8HQI1T&keywords=weatherproof+electrical+boxes&qid=1559096148&s=gateway&sprefix=weatherproof+%2Caps%2C158&sr=8-7)).

#####Not necessary but will make your life easier:
* [Breadboard](https://www.adafruit.com/product/381) (In retrospect, a smaller one is ok, but for quick prototyping and to reuse for future projects, this will do)
* [Breadboarding wire bundle](https://www.adafruit.com/product/153)
* [Assembled Pi Cobbler Plus](https://www.adafruit.com/product/2029) (When I ordered most of the items, this item was out of stock, so I ordered a [T-Cobbler plus kit](https://www.adafruit.com/product/2029)). Only used to test out wiring, though a few female/male jumpers could do the trick too if you have them.
* [Small Alligator Clip Test Lead](https://www.adafruit.com/product/4100)

##Assembly / Hardware
I'd say the difficulty level is at a 3. If you can solve those toddler puzzles where they have to insert a cube into a square hole, triangle to a triangle, etc, then you'll do great putting this together. Disclaimer: As with all things electrical, be careful. 
The scope of this tutorial is not about how to set up a Raspberry Pi, so I'm going to skip forward. I used the [Raspbian Stretch Lite](https://www.raspberrypi.org/downloads/raspbian/) distro, in case you're curious. 
Let’s start with the ribbon cable. Notice that this is a keyed ribbon cable, and there is a white strip on one side of the cable (ground). 
<img src="/content/images/2019/05/IMG_0791.jpg">

On the breadboard, place the 4.7KOhm resistor and the relevant jumper cables. Below are my pictures but follow the guides from Peter Kodermac’s tutorial (his site is [https://www.kodermac.com/](https://www.kodermac.com/) and the weather tutorial is [raspberryweather.com/wiring-ds18b20](https://www.raspberryweather.com/wiring-ds18b20/). You could just ignore this tutorial and follow his excellent instructions with pictures and clear explanations. Reference my instructions as a secondary source. He adds the temp readings to a MySQL database and Wordpress, which is where my tutorial differs.  If you want to jump to my part about Elasticsearch scroll down.

Long story short, plug the red wires to a 3.3v source, black wires to ground, and the yellow wires to any pin (one pin for the DHT22 and one pin for the DS18B20)
___

##Testing Probe Connections

Once both probes are plugged in correctly, it's time to test them out. In your Pi's terminal, we will set up the sensors first. For the DS18B20, you'll need to edit the file __/boot/config.txt__ and append __dtoverlay=w1-gpio__ at the end of the file, or simply type in the following commands:

```bash
$> echo "dtoverlay=w1-gpio" | sudo tee -a /boot/config.txt
```

I used `sudo tee -a` because `cat something >> file` isn't allowed due to permissions, so sudo tee is how I got around that. Reboot the Pi afterward.

When you're back in the terminal, type: 
```bash
$> sudo modprobe w1-gpio
$> sudo modprobe w1-therm
```

Once these are executed, then cd into **/sys/bus/w1/devices/** and you should see a directory starting with **28-\***. If you see something like **00-\*** or anything else that is not **28-\***, something is off and you might need to troubleshoot. If so, make sure your cables are connected properly (I've made this mistake too many times).

Once that **28-\*** dir is there, type this and you should see some output like this:

<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-11.43.58-PM.png" alt-text="sometxt">

Ignore all those scary hex numbers and focus on **t=24437**. This is the actual temperature in Celsius. [Fun fact, in 2019, the countries not using the metric system are Burma/Myanmar, Liberia, and the U.S.](https://qr.ae/TUfzmc)

##AM2302/DHT22
<img src="https://cdn-shop.adafruit.com/970x728/393-00.jpg">
There is a python library provided by Adafruit. [Follow Adafruit's excellent documentation on how to install the library using their python script.](https://github.com/adafruit/Adafruit_Python_DHT)

##Modifying the code

Now we can get to read temperatures from both sensors and do fun stuff. The most up-to-date code will be on [github](https://github.com/alonchis/Raspberry-Weather-DS18B20). 
I used the existing getInfo.py script and added some functionality. 
The script already provides the DHT22 readings, so I just added some code to read from the output of **/sys/bus/w1/devices/28–00000a9912d4/w1_slave**. I've copied this code from somewhere but I can remember where. I'll add it once I find it.
If you wish to jump to the one-liners of code to just run the working code, scroll below.

```python
os.system('modprobe w1-gpio')
os.system('modprobe w1-therm')

ES_INDEX = os.environ['ES_INDEX']
es = elasticsearch.Elasticsearch([os.environ['ES_URL']])

ds18b20_base_dir = '/sys/bus/w1/devices/'
ds18b20_device_folder = glob.glob(ds18b20_base_dir + '28*')[0]
ds18b20_device_file = ds18b20_device_folder + '/w1_slave'

//get your api key here "https://openweathermap.org/api"
OPENWEATHER_URL = 'https://api.openweathermap.org/data/2.5/weather?id=4893811&APPID=' + os.environ['OW_API_KEY'] + '&units=imperial'


# Try to grab a sensor reading.  Use the read_retry method which will retry up
# to 15 times to get a sensor reading (waiting 2 seconds between each retry).
humidity, temperature = Adafruit_DHT.read_retry(22, PIN)

//Comment the temp line to use Celcius
try:
    temperature = temperature * 9.0 / 5.0 + 32
except Exception:
    // in case there a sensor failure. it default to this value
    temperature = -273.15 
```

You'll need to substitute **OPENWEATHER_API_KEY** and **PIN** to their respective values. **PIN** would be whatever pin you plugged the **AM2303** sensor to (e.g. in my case, I plugged the yellow wire to pin 23, so PIN would be 23. [Here's a pinout aid](https://pinout.xyz/))

####Openweather GET request
```python
def get_outside_temp():
    resp = requests.get(OPENWEATHER_URL)
    if resp.ok:
        response = json.loads(resp.text)
        result = {'city': response['name'], 'humidity': response['main']['humidity'], 'temp': response['main']['temp']}
        return result
    else:
        return None
```
Performs a get request to openweather.com and parses through the relevant results

####Read AM2303/DHT22 temperatures
```python
# Note that sometimes you won't get a reading and
# the results will be null (because Linux can't
# guarantee the timing of calls to read the sensor).
# If this happens try again!
def get_readings_dht22():
    if humidity is not None and temperature is not None:
        print('DHT22 readings: Temp={0:0.1f}*  Humidity={1:0.1f}%'.format(temperature, humidity))
        result = "humidity = {}, temperature = {}".format(humidity, temperature)
        return result
    else:
        print('Failed to get reading. Try again!')
        sys.exit(1)
```
If the sensor is working properly, it will return the results

####Elasticsearch Insert New Document
```python
def build_and_send_payload(dht22_temp, dht22_humid, DS18B20_readings):
    result = es.index(index=ES_INDEX, doc_type='sensors', body={
        '@timestamp': datetime.datetime.utcnow(),
        'temp probe temp': DS18B20_readings,
        'DHT22 temp': dht22_temp,
        'DHT22 humidity': dht22_humid,
        'outside temp': get_outside_temp()
    })
    return result


read_temp_results = read_temp()
print(datetime.datetime.now().isoformat())
print("waterproof sensor reads ", read_temp_results)
print("DHT22 temp = ", temperature, " humidity = ", humidity)
print(build_and_send_payload(temperature, humidity, read_temp_results))
```
**\#build_and_send_payload** adds a new document to elasticsearch specified in the body, filling in the readings we've been gathering. 

###Environment Variables
Lastly, you'll need to set these environment variables.

* ES_INDEX=rpi-weather
* ES_URL=//url of docker machine running ELK
* OW_API_KEY=somekey

There are many ways to do this, like append them to **/etc/environment**, put them in your **~/.bashrc**, or export them. I recommend putting them in **/etc/environment** since when we run this as a cron job, those variables can be also loaded by cron user. 

###Testing the script

Once the script is in place and you have substituted the variables needed, we should test it out, right? Comment out the last line 
```python
print(build_and_send_payload(temperature, humidity, read_temp_results))
```

and run the script

```bash
$ python3 getInfo.py
```
<img src="/content/images/2019/05/getInfoOutput1.png">

Success! Now uncomment that line and let's set up Elasticsearch.

##The Elastic Stack
The Elastic stack (Elasticsearch, Kibana, and Logstash, ELK for short) is a platform used by a lot of people for real-time analytics. Elasticsearch is a type of database/document store, Kibana is the front end, and Logstash is a log shipper. How a typical workflow would happen is a program creates a log in the system, then Logstash picks up these logs, and ships them to Elasticsearch, which is where you could use Kibana to "massage" the data and do other cool things. I'll skip Logstash and send directly to Elasticsearch for simplicity's sake. 

To get ELK up and running will be very simple thanks to docker. There is a bit of overhead so the Pi might not be able to run it. I suggest to run it locally on another computer (or cloud service if you’re into that). I had some education credits on [digitalocean.com](http://www.digitalcean.com) so I run Ubuntu in a virtual private machine through them but you don't have to if you don't want to, any desktop or laptop will do just fine.

If you’re new to containerization, don’t worry, it can be done in very few instructions. Make sure you have Docker installed in your environment. There are different ways of installing docker depending on your OS, [MacOS](https://docs.docker.com/docker-for-mac/) and [Windows](https://docs.docker.com/docker-for-windows/install/) are both done through an executable, but [Ubuntu](https://docs.docker.com/install/linux/docker-ce/ubuntu/) is a bit more complicated. Once docker is installed, type this into the terminal:
```bash
$> docker pull sebp/elk
```

You should see something like this
<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-12.13.02-PM.png">

After it's done downloading the images, type:
<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-10.07.18-PM_o.png">

Now let's go and run it

<img src="/content/images/2019/05/Screen%20Shot%202019-05-27%20at%2011.19.16%20AM.png">

* **--name** is to give the container an easily identifiable name, instead of typing 5a68b110ece1 every time. 
* **-p 9X00:9X00**: These three -p commands tell docker to expose the container ports and tie them to our local machine. The syntax is local_machine_port:container_port
* **sebp/elk**: The name of the image we want to run.

you should see a bunch of text floating through your terminal but when you see the messages slow down in about 15 seconds, open up your favorite browser and go to [localhost:5601](http://localhost:5601). This is Kibana's default port and the one we tied out local port to. You should see:

<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-10.28.37-PM-1.png">

Click on "Explore my own", and you should be taken to a landing page. Before we delve into Kibana, let's create an index. Remember, Elasticsearch is the backend, and Kibana is the front end. All of Kibana's settings and dashboards are stored in Elasticsearch.

To create an index simply send a put request through the same computer where docker is running:

```bash
$ curl -XPUT “localhost:9200/rpi-temp"
```
<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-10.23.00-PM-2.png">

In this case, I named the index rpi-temp. Port is 9200 because that is the default port for Elasticsearch.

To perform a sanity check and make sure our index is created, run this command:
```bash
$ curl -XGET “localhost.com:9200/test-index/?pretty”
```
and you should get this back:
```
{ 
    "test-index" : {
        "aliases" : { },
        "mappings" : { },   
        "settings" : {
            "index" : {
                "creation_date" : "1551678167518",
                "number_of_shards" : "5",        
                "number_of_replicas" : "1",
                "uuid" : "sTAMz7y6R8-43VJtbWUw7Q",
                "version" : {
                    "created" : "6060099"
                },
            "provided_name" : "test-index"
            }
        }    
    }
}
```

Next, let's insert a new document. Elasticsearch uses documents for what a row would be in an ERDB. Run that python script again on the Pi (make sure to uncomment that last line)

```python
python3 getInfo.py
```
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-11.50.50-AM.png">

One more thing. Elasticsearch is easy to get up and running without much configuration, but just like you'd need to set up the schema in an ERD, we need to set up an index mapping in Elasticsearch. Don't worry, it's all done in very few clicks.

Back to Kibana, click the cog in the lower left side of the screen.
<img src="/content/images/2019/05/Screen-Shot-2019-05-26-at-10.30.29-PM-1.png">

1. Click on kibana/index patterns/ create index pattern.
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-11.56.36-AM.png">

2. Type "rpi-weather" in the index pattern name, hit next step.
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-11.59.34-AM.png">

3. Select @timestamp on time Filter Field Name.
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.01.09-PM.png">

4. Click Create Index Pattern and then you should see this screen:
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.01.37-PM.png"> 

5. Click on the sidebar compass icon "Discover" and you should be brought to this screen. You should see at least one record for our previous run getInfo.py. If you don't, maybe its the time frame. To the side of the blue "refresh" button, there is a time filter. Click the little calendar icon and adjust time accordingly. Here I set it up to the entire day so far

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.26.32-PM.png">

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.29.39-PM.png">


Before we move on to visualizations, we should start automatically put in new data into Elasticsearch.

##The CRON job

Now that we have a working python script, an instance of Elasticsearch and Kibana, all we need is to keep feeding it data. I’m sure there are a lot of ways, probably adding it as a systemd service and running that on a schedule would be the right way, but I prefer an easier way.
Simply add it as a cronjob!
```bash 
$ crontab -e
```

regardless of what text editor you choose, add this line at the bottom

```bash 
*/5 * * * * OW_API_KEY="some value" ES_URL="IPofComputerRunningELK:9200" ES_INDEX="name of index we created a while ago, like rpi-temp" $(which python3) /home/pi/Raspberry-Weather-DS18B20/getInfo.py >> /home/pi/pytemp.log 2>&1**
```

There a lot to explain here:

* **\*/5 * * * \*** tells cron to run it every five minutes (*/5). I found it useful to set this to every minute (* or */1 if you want to be explicit) for testing purposes, but change it back to 5 minutes or however frequent you'd like.

* **OW_API_KEY**: Is the API key I used to add the weather from openweathermaps.org just for fun. You don't need to include this line if you don't want to fetch it.

* **ES_INDEX**: The IP address or server hostname of wherever you’re running the ELK stack. For example, if my ELK stack is running in 192.168.0.101, this value would be “192.168.0.101:9200” (no quotes). 
    * to find out your laptop/computer's IP address, on Linux: `hostname -I`, on Windows you can find it on network interfaces, and viewing properties of the interface you're using.

* **\$(which python3)**: Cron likes absolute addresses. The `which` command would substitute the location of python3. You could also hardcode it if you want.

* **/home/$USER/Raspberry-Weather-DS18B20/getInfo.py**: This is the location of the python script. If you cloned the git repo in your home directory, that’s what you would type. If you installed it in some other folder be sure to include the full path.

* **/home/pi/pytemp.log 2>&1**: >> means append the output of the previous command. Append means that if, for example, the file already contains some text, it will simply add the newer text at the bottom of the file. If you type a single > it will append the output and overwrite everything in that file. 2>&1 means redirects both stderr and stdout to wherever the output is already going. [This post explains by Mathias Bynensit much better than I can.](https://stackoverflow.com/questions/876239/how-can-i-redirect-and-append-both-stdout-and-stderr-to-a-file-with-bash#876242)

==important note==: because of a weird issue where cron is run under a different user, it would be cleaner to globally set env vars by appending them to **/etc/environment**. I just put them in the same cron command to make this guide easy.

Now we can expect the measurement readings to be sent to ELK every minute (don't forget to change it later on. you could keep it at 1 minute but IMO this is the point of diminishing returns of whatever information you can get from knowing the temperature minute by minute. 5 minutes seems like a decent interval for me, but this is your project! You can do whatever you’d like)

##Creating dashboards in Kibana
In Kibana, go to visualize/new visualization, and select the rp* index, and fill in this information. Once typed in, click the blue triangle icon at the top right, and you should see your data points show up on the graph on the right. Once this works, save the visualization with whatever name you like.
Once saved, in the same screen, change the Y-axis field to whatever your temperature sensor point is called, and change the label to average temperature. save the visualization as a new visualization and change the name to temperatures or the like.

1. Click the fourth icon top to bottom on the sidebar for Dashboards. On the next screen that shows up, click "Create New Dashboard"
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.55.59-PM.png">

2. Click Add on the top menu bar  / "Add New Visualization" / Line / Select the rpi-weather index we just created. 
<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.53.05-PM.png">

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-12.59.33-PM.png">

3. Now we tell it what to plot. Under metrics click the Y-axis and fill in this info for all three data points. We'll be adding three y-axis metrics by clicking "add metrics":
   1. DHT22 temps
      1. Aggregation: Average
      2. Field: DHT22 Temp
      3. Custom Label: DHT22 temp
   2. DS18B20 temps
      1. aggregation: Average
      2. Field: temp probe temp
      3. Custom Label: DS18B20 temps
   3. Openweather temps:
      1. aggregation: Average
      2. Field: outside temp.temp
      3. Custom Label: OW temps
   4. Under Buckets; X-Axis
      1. aggregation: date histogram
      2. field: @timestamp
      3. interval: auto

Once you're done filling in the fields, click on the blue arrow-shaped icon to apply the changes. Like all cooking shows, I gave you all instructions on how to cook the dish, but at the time of putting stuff in the oven, they pull out an already made version of the dish. Well, I went ahead in advance and started recording data from about a week ago. This is how the chart will look eventually.

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-1.21.32-PM.png">

Hit Save at the top menu bar and name it something descriptive like "temperatures"

repeat the same steps but this time for humidity from the sensor and openweather's data. Save it and the visualization will be added to the dashboard.

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-1.29.36-PM.png">

There's a lot of possibilities. This is a quick and dirty dashboard I put together. 

<img src="/content/images/2019/05/Screen-Shot-2019-05-27-at-1.40.35-PM.png">

There can be many other cool inferences to be made. For example, in my area, our electrical provider has an API available for the current power rate. Adding that to getInfo.py, I could make another visualization comparing the temperature and the power rate. etc. 

It would also be cool to check out the Kibana sample dashboards and try to have a quick sample of all the things that Kibana/elastic is capable of. 

I Hope this was informative and straight forward. I'm not a writer but I wanted to in a way, return the favor to all the tutorials that have helped me throughout my free time. Feel free to reach out if you have questions! Happy hacking.
