# MQTT-images-transfer-and-receiver-from-multiple-raspberry-pis
MQTT-images-transfer-and-receiver-from-multiple-raspberry-pis


### Project Structure

Here is the structure for your GitHub repository:

```
MQTT-images-transfer-and-receiver-from-multiple-raspberry-pis/
├── publisher_pi1.py
├── publisher_pi2.py
├── mqtt_image_receiver.py
├── requirements.txt
├── prerequisites.txt
├── README.md
```

### Publisher Script (For Raspberry Pi 1 and 2)

**publisher_pi1.py** (same for `publisher_pi2.py`, just change the topic to `images/pi2`):

```python
import os
import logging
import paho.mqtt.client as mqtt
import time
import base64

# Configuration
broker = 'broker.hivemq.com'
port = 1883
topic = 'images/pi1'  # Change to 'images/pi2' for pi2 publisher
image_counter_file = 'image_counter.txt'
image_directory = './'
processed_folder = 'processed_images'

# Set up logging
logging.basicConfig(filename='image_capture_mqtt.log', level=logging.INFO,
                    format='%(asctime)s %(levelname)s:%(message)s')

# MQTT callbacks
def on_connect(client, userdata, flags, rc):
    logging.info(f"Connected with result code {rc}")
    print(f"Connected with result code {rc}")

def on_publish(client, userdata, mid):
    logging.info(f"Message published with mid {mid}")

# Capture image
def capture_image(image_path):
    try:
        os.system(f'fswebcam -r 1280x720 --no-banner {image_path}')
        logging.info(f"Image captured and saved to {image_path}")
    except Exception as e:
        logging.error(f"Failed to capture image: {e}")

# Publish image
def publish_image(client, image_path):
    try:
        # Read and encode image
        with open(image_path, 'rb') as file:
            image_data = base64.b64encode(file.read()).decode()

        # Publish encoded image
        result = client.publish(topic, image_data, qos=1)
        if result.rc == mqtt.MQTT_ERR_SUCCESS:
            logging.info(f"Image {image_path} published successfully.")
        else:
            logging.error(f"Failed to publish image {image_path}. Error code: {result.rc}")

        # Move file to processed folder
        os.makedirs(processed_folder, exist_ok=True)
        os.rename(image_path, os.path.join(processed_folder, os.path.basename(image_path)))

    except Exception as e:
        logging.error(f"Failed to publish image: {e}")

# Get next image number
def get_next_image_number(counter_file):
    try:
        with open(counter_file, 'r') as file:
            number = int(file.read().strip())
    except FileNotFoundError:
        number = 1
    return number

# Update image number
def update_image_number(counter_file, number):
    with open(counter_file, 'w') as file:
        file.write(str(number))

# Main function
def main():
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_publish = on_publish

    client.connect(broker, port, 60)
    client.loop_start()

    while True:
        try:
            image_number = get_next_image_number(image_counter_file)
            image_path = os.path.join(image_directory, f"image_{image_number:04d}.jpg")

            capture_image(image_path)
            publish_image(client, image_path)

            image_number += 1
            update_image_number(image_counter_file, image_number)

            time.sleep(10)
        except KeyboardInterrupt:
            logging.info("Keyboard interrupt detected. Stopping the script.")
            break
        except Exception as e:
            logging.error(f"Unexpected error occurred: {e}")
            time.sleep(60)

    client.loop_stop()
    client.disconnect()

if __name__ == "__main__":
    main()
```

### Receiver Script (For Jetson Nano)

**mqtt_image_receiver.py**:

```python
import os
import logging
import paho.mqtt.client as mqtt
import base64

# Configuration
broker = 'broker.hivemq.com'
port = 1883
image_topic_pi1 = 'images/pi1'
image_topic_pi2 = 'images/pi2'
image_directory_pi1 = './images_from_pi1/'
image_directory_pi2 = './images_from_pi2/'

# Set up logging
logging.basicConfig(filename='logs/image_receiver.log', level=logging.INFO,
                    format='%(asctime)s %(levelname)s:%(message)s')

# MQTT callbacks
def on_connect(client, userdata, flags, rc):
    logging.info(f"Connected with result code {rc}")
    logging.info(f"Subscribing to topics {image_topic_pi1} and {image_topic_pi2}")
    client.subscribe(image_topic_pi1)
    client.subscribe(image_topic_pi2)

def on_message(client, userdata, msg):
    try:
        topic = msg.topic
        image_data = base64.b64decode(msg.payload)
        
        # Determine the correct directory and prefix based on the topic
        if topic == image_topic_pi1:
            image_directory = image_directory_pi1
            prefix = 'p1_'
        elif topic == image_topic_pi2:
            image_directory = image_directory_pi2
            prefix = 'p2_'
        else:
            logging.error(f"Unknown topic: {topic}")
            return
        
        # Get the next image number
        image_number = get_next_image_number(image_directory, prefix)
        image_path = os.path.join(image_directory, f"{prefix}{image_number:02d}.jpg")

        # Save the image
        os.makedirs(image_directory, exist_ok=True)
        with open(image_path, 'wb') as file:
            file.write(image_data)
        
        logging.info(f"Image received and saved to {image_path}")

    except Exception as e:
        logging.error(f"Failed to process message: {e}")

def get_next_image_number(directory, prefix):
    # Get the list of files in the directory
    files = os.listdir(directory)
    # Extract image numbers from file names
    numbers = [int(f.split('_')[1].split('.')[0]) for f in files if f.startswith(prefix) and f.split('_')[1].split('.')[0].isdigit()]
    return max(numbers, default=0) + 1

def main():
    logging.info("Starting MQTT image receiver...")
    client = mqtt.Client()
    client.on_connect = on_connect
    client.on_message = on_message

    client.connect(broker, port, 60)
    client.loop_forever()

if __name__ == "__main__":
    main()
```

### Requirements File

**requirements.txt**:

```
paho-mqtt==1.5.1
```

### Prerequisites File

**prerequisites.txt**:

```
1. Ensure that you have Python 3 installed on all devices.
2. Install `fswebcam` on the Raspberry Pi devices to capture images:
   sudo apt-get install fswebcam
3. Install the required Python libraries:
   pip install -r requirements.txt
4. Set up the directories for saving images on the Jetson Nano:
   mkdir -p images_from_pi1
   mkdir -p images_from_pi2
   mkdir -p logs
```

### README File

**README.md**:

```markdown
# MQTT Images Transfer and Receiver from Multiple Raspberry Pis

This project demonstrates how to capture images from multiple Raspberry Pis and send them to a Jetson Nano via MQTT. The images are saved in different folders based on the originating Raspberry Pi.

## Prerequisites

1. Ensure that you have Python 3 installed on all devices.
2. Install `fswebcam` on the Raspberry Pi devices to capture images:
   ```bash
   sudo apt-get install fswebcam
   ```
3. Install the required Python libraries:
   ```bash
   pip install -r requirements.txt
   ```
4. Set up the directories for saving images on the Jetson Nano:
   ```bash
   mkdir -p images_from_pi1
   mkdir -p images_from_pi2
   mkdir -p logs
   ```

