# Nutrition-Prediction-Model
from sklearn.ensemble import RandomForestRegressor
from sklearn.metrics import mean_squared_error
from sklearn.model_selection import train_test_split
import pandas as pd
import serial
import time

import RPi.GPIO as GPIO

GPIO.setmode(GPIO.BCM)

import Adafruit_SSD1306

from PIL import Image
from PIL import ImageDraw
from PIL import ImageFont
from firebase import firebase




def button_pressed_callback(channel):
    global n,p,k,flag,mg,S,ca
    flag=False
    disp.clear()
    draw.rectangle((0,0,width,height), outline=0, fill=0)
    print("Prediction running")
    
    new_data = pd.DataFrame({'L': [1], 'N': [n], 'P': [p],'K':[k]})
    new_pred = rf_model.predict(new_data)
    print('New Predictions:', new_pred)

    mg = new_pred[0][0]
    S = new_pred[0][1]
    ca = new_pred[0][2]

    print(mg,S,ca)
    draw.text((x, top),   "Predicted Nutritions",  font=font, fill=255)
    draw.text((x, top+15),  "Mg: "+str(mg), font=font, fill=255)
    draw.text((x, top+30),  "S: "+str(S),  font=font, fill=255)
    draw.text((x, top+45),   "Ca: "+str(ca),  font=font, fill=255)
    disp.image(image)
    disp.display()
    time.sleep(8)
    flag=True
    


def main():
    global n,p,k,flag,mg,ca,S
    print('Running. Press CTRL-C to exit.')
    with serial.Serial("/dev/ttyACM0", 9600, timeout=1) as arduino:
        time.sleep(0.1)
        if arduino.isOpen():
            print("{} connected!".format(arduino.port))
            try:
                while True:
                    

                    cmd="data"
                    arduino.write(cmd.encode())
                    time.sleep(1)
                    print("Accessing")
                    

                    if  arduino.inWaiting()>0: 
                        answer=str(arduino.readline())
                        print("---> {}".format(answer))
                        if cmd=="data":
                            dataList=answer.split("x")
                            print(dataList)
                            
                            ph_str = dataList[0]
                            temp = dataList[2]
                            turbidity = dataList[3]
                            n  = dataList[4]
                            p  = dataList[5]
                            k_str  = dataList[6]
                            
#                             ph = float(ph_str[2:])
                            ph = 8.762
                            humidity = float(dataList[1])
                            k = int(k_str[:-1])
                            
                            

                            print("Ph : {}".format(ph))
                            print("Humidity : {}".format(humidity))
                            print("Temperature: {}".format(temp))
                            print("Turbidity: {}".format(turbidity))
                            print("N: {}".format(n))
                            print("P: {}".format(p))
                            print("K: {}".format(k))
                            
                            firebase.put('/Sensors','ph',ph)
                            firebase.put('/Sensors','Humidity',humidity)
                            firebase.put('/Sensors','Temperature',temp)
                            firebase.put('/Sensors','Turbidity',turbidity)
                            firebase.put('/Sensors','N',n)
                            firebase.put('/Sensors','P',p)
                            firebase.put('/Sensors','K',k)
                            
#                             firebase.put('/Sensors','S',S)
#                             firebase.put('/Sensors','Ca',ca)
#                             firebase.put('/Sensors','Mg',mg)
                            
                            if flag:
                                draw.rectangle((0,0,width,height), outline=0, fill=0)
                                draw.text((x, top),   "PH: "+str(ph),  font=font, fill=255)
                                draw.text((x, top+10),  "Temperature: "+str(temp), font=font, fill=255)
                                draw.text((x, top+20),  "Turbidity: "+str(turbidity),  font=font, fill=255)
                                draw.text((x, top+30),   "N:"+str(n),  font=font, fill=255)
                                draw.text((x, top+40),   "P:"+str(p),  font=font, fill=255)
                                draw.text((x, top+50),   "K:"+str(k),  font=font, fill=255)
                                
                                disp.image(image)
                                disp.display()
                            time.sleep(.1)
                            

                            
                            
                            
                            arduino.flushInput()
                        
                        time.sleep(1)
                    time.sleep(0.5)
                            
            except KeyboardInterrupt:
                print("KeyboardInterrupt has been caught.")


if __name__ == '__main__':
    global n,p,k,flag,mg,S,ca
    firebase =firebase.FirebaseApplication('https://hydro-f4287-default-rtdb.asia-southeast1.firebasedatabase.app/',None)
    flag=True
    disp = Adafruit_SSD1306.SSD1306_128_64(rst=None)

    buttonPin = 21
    GPIO.setup(buttonPin, GPIO.IN, pull_up_down=GPIO.PUD_UP)
    
    GPIO.add_event_detect(buttonPin, GPIO.FALLING, 
                callback=button_pressed_callback, bouncetime=1000)

    
    disp.begin()

    
    disp.clear()
    disp.display()

    
    width = disp.width
    height = disp.height
    image = Image.new('1', (width, height))

    
    draw = ImageDraw.Draw(image)

    
    draw.rectangle((0,0,width,height), outline=0, fill=0)

    
    padding = -2
    top = padding
    bottom = height-padding
    
    x = 0


    
    font = ImageFont.truetype("/usr/share/fonts/truetype/freefont/FreeMonoBold.ttf", 12)

    
    data = pd.read_csv('12.csv')

    
    X = data[['L', 'N', 'P','K']]

    
    y = data[['Mg', 'S', 'Ca']]

    
    X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

    
    rf_model = RandomForestRegressor(n_estimators=100, random_state=42)

    
    rf_model.fit(X_train, y_train)

    
    y_pred = rf_model.predict(X_test)
    
    main()


