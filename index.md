# Material Sorting Robot
This project is a robotic arm that uses object detection to distinguish between and sort items of different materials. The arm uses Google's Coral Edge TPU to do this. 

| **Engineer** | **School** | **Area of Interest** | **Grade** |
|:--:|:--:|:--:|:--:|
| Nandini R. | Lynbrook High School | Computer Engineering | Incoming Junior

headstone image goes here
  
# First Milestone
<p>My first milestone was setting up my Raspberry Pi and building and controlling my robotic arm. The parts for the robotic arm came in a kit. To build it, I just assembled the provided parts. The grabbing attachment could be attached horizontally or vertically on the arm, so I decided to keep it vertical so that it would be easier to grab items on the ground. The  arm uses 4 servo motors - 1 to control the base, 2 to control the arm, and 1 to control the grabbing attachment.</p> <p>After assembling the arm, I wired the servo motors. Each servo motor is connected to 0V, 5V, and a GPIO pin. A diagram of the wiring can be found below. When I finished wiring, my next step was to program the arm to pick an object up, move, and drop it somewhere else. You can find the code I used to program the arm to do this below. My next step will be to train a machine learning model so that eventually, the robot's movements will depend on what material it recognizes in front of it, in contrast to the current hard coded software.</p> 


**Code to control the robotic arm:** 
```python
from gpiozero import Servo
from time import sleep
base = Servo(4) #servo controlling the base
horizontal = Servo(17) #servo on the arm controlling horizontal movement
vertical = Servo(18) #servo on the arm controlling vertical movement
gripper = Servo(27) #servo controlling the gripping attachment 
try:
    while True:
        base.mid() #moves arm to the front
        sleep(0.5)
        horizontal.mid() #extends arm 
        sleep(0.5)
        vertical.min() #lowers arm
        sleep(0.5)
        gripper.min() #picks up object
        sleep(0.5)
        vertical.max() #lifts arm
        sleep(0.5)
        base.max() #moves robot around its base to a different position
        sleep(0.5)
        vertical.min() #lowers arm
        sleep(0.5)
        gripper.max() #releases object 
        sleep(0.5)
        
except KeyboardInterrupt:
    print("Program stopped")
```

**Circuit Diagram:**

![image](circuit.png)


<!-- milestone video here-->
**Milestone 1 Video** 
<iframe width="560" height="315" src="https://www.youtube.com/embed/OB5t3VPEeiY" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>


# Second Milestone
<p>My second milestone was training my object detection model. I decided to use a pretrained model, the SSD MobileNet V2, to do this. I started by finding a dataset which worked for me, and found this one: https://github.com/bandofpv/Trash_Sorting_Robot. This dataset has over 2500 images of glass, trash, plastic, paper, metal, and other such materials.</p> <p> Training the model took me multiple attempts. My first try was to train the model using the TFRecord format. The TFRecord format is an easier way to store a sequence of binary records. I started by preprocessing my data. First, I cloned the github with the data I was using, and then loaded the CSV file, like so: </p>
  
  ```python
  ! git clone https://github.com/bandofpv/Trash_Sorting_Robot.git 
  csv_file = 'CSV/train_labels.csv'
  csv = pd.read_csv(csv_file)
  csv
  ```
 
After that, I wrote a function to preprocess my data. It uses information from the csv file such as the image name and label to resize and reformat the images so that they work with the SSD Mobilenet V2 model that I wanted to use. 
```python
def saveData(source, filename, path, width=350, height=350):
  # Data structure
  data = dict()
  data['filename'] = []
  data['class'] = []
  data['image'] = []
  data['bnd box'] = {
      'xmin' : [],
      'ymin' : [],
      'xmax' : [],
      'ymax' : []
  }

  pkl = f'{filename}_{width}x{height}px.pkl'
  tfrecord_file_name = f'{filename}_{width}x{height}px.tfrecord'
  csv = pd.read_csv(path)

  # PREPROCESSING
  for folder in os.listdir(source): # ./
    if folder == 'Images': # /Images/
      curr_path = os.path.join(source, folder) # ./Images/
      for images in tqdm(os.listdir(curr_path + '/train/')):
        if images[-3:] == 'jpg':
          print("in here")
          img = cv2.imread(curr_path + '/train/' + images,0)
            #img = resize(img, (width, height))
          b = bytearray(img)
          b = bytes(b)
        
          #img = cv2.imread(curr_path + '/train/' + images) # example: ./Images/train/plastic91.jpg

          index = csv.index[csv['filename']==images][0]
          # data['filename'].append(images)
          # data['class'].append(csv.iloc[index]['class'])
          # data['image'].append(img)
          # data['bnd box']['xmin'].append(csv.iloc[index]['xmin']) 
          # data['bnd box']['ymin'].append(csv.iloc[index]['ymin'])
          # data['bnd box']['xmax'].append(csv.iloc[index]['xmax'])
          # data['bnd box']['ymax'].append(csv.iloc[index]['ymax'])
          std_slc = StandardScaler() 
          x_std = std_slc.fit_transform(img)
          pca = decomposition.PCA(n_components=4)
          x_std_pca = pca.fit_transform(x_std)
          example = tf.train.Example(features = tf.train.Features(feature = {
            'filename': tf.train.Feature(bytes_list = tf.train.BytesList(value = [bytes(images, 'utf-8') ])),
            'Image shapes': tf.train.Feature(int64_list = tf.train.Int64List(value = [width,height])),
            'image': tf.train.Feature(bytes_list = tf.train.BytesList(value = [b])),
            'label': tf.train.Feature(bytes_list = tf.train.BytesList(value = [bytes(csv.iloc[index]['class'], encoding = 'utf8')])),
            'features': tf.train.Feature(bytes_list = tf.train.BytesList(value = [bytes(x_std_pca)])) }))

    
      break
  with tf.io.TFRecordWriter(tfrecord_file_name) as writer:
    writer.write(example.SerializeToString()) 
  # f = open(pkl, "wb")
  # pickle.dump(data, f)
  print("done")
  # f.close()
  ```
This code went through quite a few iterations, which is why there are many things that I ended up commenting out. The next step for me was to extract the data, but I wasn't able to find a way to do this. So, I decided to completely scratch everything I did in the above code and find another way to train my model. 
<p> Once again, I cloned the github, and then preprocessed the data. However, I did the preprocessing in a different way this time: </p>
```
model_tpu = 'Trash_Sorting_Robot/RPI/recycle_ssd_mobilenet_v2_quantized_300x300_coco_2019_01_03/detect_edgetpu.tflite'
model= 'detect.tflite'
interpreter = tf.lite.Interpreter(model)
interpreter.allocate_tensors()
inputdetails = interpreter.get_input_details()
inputdetails
outputdetails = interpreter.get_output_details()
outputdetails
dir = 'Trash_Sorting_Robot/Tensorflow/Images/test/'
for image in os.listdir(dir):
  if image[-3:] == 'jpg':
    input_shape = inputdetails[0]['shape']
    input_data = cv2.imread(dir + image)
    input_data = cv2.resize(input_data, (300, 300))
    input_data = (1, ) + input_data + (3, )
    input_data = input_data.astype('uint8')
    input_data = np.expand_dims(input_data, 0)
    interpreter.set_tensor(inputdetails[0]['index'], input_data)

    interpreter.invoke()

    # The function `get_tensor()` returns a copy of the tensor data.
    # Use `tensor()` in order to get a pointer to the tensor.
    output_data = interpreter.get_tensor(outputdetails[-1]['index'])
    print(image, output_data)
 ```
The above code preprocesses the data and inputs it into the model. Unfortunately, this wasn't able to run on my Raspberry Pi. I decided to just use the trained model that the original creator of this project made (available at the github link) and tested it out on the Raspberry Pi, with the intention of going back and using my model later on after debugging. 
<!-- milestone video here-->
milestone video coming soon 
<img src = "https://mir-s3-cdn-cf.behance.net/project_modules/disp/35771931234507.564a1d2403b3a.gif">
# Final Milestone
  

My final milestone was integrating my object detection model to the Raspberry Pi and using it to control my robotic arm. 

<!-- milestone video here-->
milestone video coming soon 
<img src = "https://mir-s3-cdn-cf.behance.net/project_modules/disp/35771931234507.564a1d2403b3a.gif">
