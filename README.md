# Dronebot-Imigorgonzola
import numpy as np
import mss
import math
import socket



HOST = '192.168.2.3'  
PORT = 65432




def rotate(power, gradi):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:   
        s.connect((HOST, PORT))
        comando = f"ROTATE:{power}:{gradi}"
        s.sendall(comando.encode())
        data = s.recv(1024)
        print("Rover:", data.decode())


def go_forward(power, gradi):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))
        comando = f"FORWARD:{power}:{gradi}"
        s.sendall(comando.encode())
        data = s.recv(1024)
        print("Rover:", data.decode())

def stop():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))
        comando = "STOP"
        s.sendall(comando.encode())
        data = s.recv(1024)
        print("Rover:", data.decode())






marker_riconoscibili = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_4X4_50)    
regole_di_riconoscimento = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(marker_riconoscibili, regole_di_riconoscimento)

catturatore_schermo = mss.mss()            
monitor = catturatore_schermo.monitors[1]

angolo_precedente = 0
xc_precedente = 0
yc_precedente = 0
n = 0


   
model = YOLO("best.pt")        



cv2.namedWindow("Fire Detection - Screen", cv2.WINDOW_NORMAL)  

while True:
    screenshot = catturatore_schermo.grab(monitor)
    frame = np.array(screenshot)
    frame = cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)

    results = model(frame)      
    annotated_frame = results[0].plot()

    annotated_frame = cv2.resize(annotated_frame, (800, 600))

    cv2.imshow("Fire Detection - Screen", annotated_frame)

    key = cv2.waitKey(1) & 0xFF
    if key == 32:
        break

cv2.destroyAllWindows()


cv2.namedWindow("Guida rover", cv2.WINDOW_NORMAL)
rotation = False

while True:
   
        screenshot = catturatore_schermo.grab(monitor)
        frame = np.array(screenshot)
        frame = cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        corners, ids, rejected = detector.detectMarkers(gray)

        if ids is not None:
            print("Marker trovato:", ids.flatten())  
            cv2.aruco.drawDetectedMarkers(frame, corners, ids)
            marker = corners[0][0]
            x4, y4 = marker[0]
            x3, y3 = marker[1]
            x2, y2 = marker[2]
            x1, y1 = marker[3]
            dx = x2-x1
            dy = y2-y1
           
            angolo = math.atan2(dy, dx)
            angolo_deg = angolo*180/math.pi
            print(angolo_deg)
            delta_angolo = angolo_deg - angolo_precedente
            delta_angolo = (delta_angolo + 180) % 360 - 180
           
            xc = (x1 + x2 + x3 + x4) / 4
            yc = (y1 + y2 + y3 + y4) / 4
            delta_xc = xc - xc_precedente
            delta_yc = yc - yc_precedente
            xc_frame = frame.shape[1] / 2
            yc_frame = frame.shape[0] / 2 - 20
            distanza_marker = np.sqrt((xc_frame - xc) ** 2 + (yc_frame - yc) ** 2)
            lato = np.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2)
            pixel = 20.5 / lato
            print(distanza_marker * pixel)
            arco = abs(angolo) * 5.9

            if abs(delta_angolo) <= 5 and abs(delta_xc) <= 5 and abs(delta_yc) <= 5:
                n += 1
            else :
                n = 0

            if (n >= 6 and angolo_deg < 0):
                rotate(10, (arco / (5.5 * np.pi)) * 360)
                n = 0

            if (n >= 6 and angolo_deg > 0):
                rotate(-10, (arco / (5.5 * np.pi)) * 360)
                n = 0
            
                

            
               
            if(abs(angolo_deg) < 7 and n >= 4):
                go_forward(50, ((distanza_marker * pixel)/(np.pi * 5.5)) * 360)
                
               
           
           
           
           
               
                   
           
           
            angolo_precedente = angolo_deg
            xc_precedente = xc
            yc_precedente = yc
           
           
     
   
        else:
            n = 0
            print("Nessun marker trovato")

        cv2.imshow("Guida rover", frame)
        cv2.waitKey(1)
