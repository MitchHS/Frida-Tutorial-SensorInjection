# Frida Tutorial: Android Sensor Injection

These days its quite common for android malware to perform a number of anti-emulation techniques to deter dynamic malware analysis. One of the more advanced techniques is performing calculations on sensor values returned from the accelerometer, gyroscope or GPS data. A good example of this is the Anubis malware. 

For example, lets say we want a malicious function to occur when our phone is 'accelerating'. Our accelerometer has 3 axis points of information, from there is quite easy to calculate a formula to determine whether our device is moving realistically or not.

This tutorial will document some basic frida techniques to bypass sensor injection by overloading android source code functions within our taget process. It may be easier to overload our specific function to return true, but for automation purposes overloading our sensor returns makes life easier for dynamic analysis. 




## Basic frida script setup

For every frida script this will generally be your basic format. I won't cover how to set up your AVD with frida-server or adb commands as that is well documented elsewhere. In this example I'll be using a test APK that I made that is located in the /APK/ of this project. 

###### fridaScript.py
```
import frida
import os
import sys
import time


def on_message(message, data):
    print(message)


device = frida.get_usb_device()
pid = device.spawn(["com.example.sensor"])
session = device.attach(pid)
f = open("C:/Path2Script")
script = session.create_script(f.read())
script.on('message', on_message)
script.load()
time.sleep(1)  # fails without this sleep
device.resume(pid)
sys.stdin.read()
```


###### Sensors.js
```
Java.perform(function() {
send("Smokescreen!")



});


```

###### Sensor apk

This APK I developed from scratch to test the accelerometer, gyro and gps values. Deveoping your own apps for frida gives you a deeper understanding of how the internals of android source code works. I would encourage anyone learning frida with android to atleast learn the fundamentals of developing your own app. Android dev documentation is quite good, so it's not hard for beginners. (Assuming you know how to code already)

For the scope of this tutorial we will only look at motion sensor data manipluation, however the app handles gps. This can be done on your own time if you're interested! 

Here is the usage for sensors according to android: https://developer.android.com/guide/topics/sensors/sensors_motion#java

My accelerometer and gyro source code snippets for this application. Worth opening up the APK in JADX-GUI to understand the full scope. 
 
```

  sensorManager = (SensorManager) getSystemService(SENSOR_SERVICE);
  gyroSensor = sensorManager.getDefaultSensor(Sensor.TYPE_GYROSCOPE);
  accelSensor = sensorManager.getDefaultSensor(Sensor.TYPE_ACCELEROMETER);
  sensorManager.registerListener(this, accelSensor, SensorManager.SENSOR_DELAY_NORMAL);
  
  
   int counter = 0;
    @Override
    public final void onSensorChanged(SensorEvent event) {

        sensorManager = (SensorManager) getSystemService(Context.SENSOR_SERVICE);

        float x = event.values[0];
        float y = event.values[1];
        float z = event.values[2];
        String xAxis = event.sensor.getName() +"\nX axis: " + String.valueOf(x) + "\n";
        String yAxis = "Y axis: " + String.valueOf(y) + "\n";
        String zAxis = "Z axis: " + String.valueOf(z) + "\n";

        TextView text = findViewById(R.id.textBuild);
        text.setText(xAxis + yAxis + zAxis);


        if(counter == 100){sensorManager.unregisterListener(this, accelSensor);}
        if(counter == 200){sensorManager.registerListener(this, accelSensor, SensorManager.SENSOR_DELAY_NORMAL);}
        if(counter == 250){sensorManager.unregisterListener(this, accelSensor);}
        
        // Automatically deregister listeners on event count and re-register. Useful for testing hooks within the register method.

        counter++;

    }


```

## Diving into Android

Based on the code snippet  ``` sensorManager = ( SensorManager) getSystemService(SENSOR_SERVICE); ``` we can determine that we should take a peak into the SensorManager service. The most valuable source for Frida in Android is the SDK source code which can be found here: 


> https://github.com/AndroidSDKSources - Find your AVD SDK. 




Doing some research on the Android Dev website we can find information on how Android handles sensor events. 


> https://developer.android.com/reference/android/hardware/SensorEvent

From this information we can assume that there must be a dispatcher for handling events. Doings a quick search of the repository for Sensor Manager returns the class android/hardware/SystemSensorManager.java. Lets take a look at the function registerListener to get an idea whether we're in the right spot. Working with Android anytime I see 'Impl' on the end of a function, it tends to be the one used to pass data. So lets start there. 


```
var systemSensorManager = Java.use('android.hardware.SystemSensorManager');
     systemSensorManager.registerListenerImpl.overload('android.hardware.SensorEventListener', 'android.hardware.Sensor',
        'int', 'android.os.Handler', 'int', 'int').implementation = function(listener, sensor, delay, handler, Latency, reservedFlags) {
         
          if(sensor != null){
          send("Hooked for sensor: " + sensor)
          }

         return this.registerListenerImpl.call(this, listener, sensor, delay, handler, Latency, reservedFlags);
        }

```


Using this script we get the output: 

> {'type': 'send', 'payload': 'Smokescreen!'}
{'type': 'send', 'payload': 'Hooked for sensor: {Sensor name="Goldfish 3-axis Accelerometer", vendor="The Android Open Source Project", version=1, type=1, maxRange=2.8, resolution=2.480159E-4, power=3.0, minDelay=10000}'}




