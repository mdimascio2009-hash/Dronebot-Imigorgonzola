# Dronebot-Imigorgonzola

# descizione tecnica

Il nostro piano ha inizio con il decollo del drone dalla posizione di partenza verso l'obiettivo venendo pilotato dall'operatore. Durante il volo, il drone utilizza la rete neurale YOLO per scansionare il territorio e identificare il fuoco. Questo modello di riconoscimento viene addestrato tramite un dataset specifico e integrato nel codice di controllo utilizzando la libreria ultralytics. Una volta confermata la presenza del fuoco, il drone rientra temporaneamente alla postazione iniziale per avviare la guida del rover. Il drone guida il rover (modello LEGO Mindstorms EV3) dall'alto fino al fuoco. Per stabilire una comunicazione visiva precisa, sul rover è posizionato un ArUco marker che consente di calcolare sia l’angolo di rotazione sia la distanza tra rover e drone: attraverso l’analisi dei vertici del marker nel frame video, il sistema determina la posizione esatta del rover convertendo i pixel in metri, mentre il coefficiente angolare del marker viene utilizzato per definire l’orientamento del rover rispetto al dorne. Ricevuti questi dati, il rover ruota per allinearsi al drone e avanza lungo una linea retta fino a posizionarsi al di sotto del dorne. Questo processo di rilevamento, calcolo e movimento viene ripetuto ciclicamente finché il robot non raggiunge il punto esatto del fuoco. L'intero sistema è coordinato da un PC centrale. Il flusso video del drone passa attraverso uno smartphone tramite l'app DJI e viene proiettato sul computer con il software scrcpy. Sul PC, la libreria Python mss cattura costantemente screenshot dello schermo per elaborare i dati e riconoscere i marker o le fiamme. Il rover, equipaggiato con il sistema operativo ev3dev, riceve i comandi di movimento dal PC tramite una connessione SSH via Bluetooth. Per poter programmare i movimenti del rover con python, abbiamo dovuto cambiare il sistema operativo del rover con una micro sd al cui interno abbiamo inserito il sistema operativo ev3dev. Invece per fare in modo che i comandi arrivassero direttamente dal codice python sul computer abbiamo installato l’app MobaXterm, che funge da collegamento tra il codice e il rover. 

# punti di forza

Il nostro progetto si distingue per due punti di forza principali: semplicità e agilità. Il rover, dotato di sole due ruote, rappresenta un grande vantaggio perché può ruotare su sé stesso con precisione, orientandosi rapidamente nella direzione indicata dal drone senza dover eseguire manovre complesse. Inoltre, l’approccio adottato è lineare e intuitivo: richiede pochi passaggi, rendendo il codice più fluido, leggero e facilmente eseguibile.

# innovazioni

# flowchart

A. Inizio:decollo drone
B. ricerca fuoco con la rete neurale YOLO
C. fuoco trovato?    C1. no > B   C2. si > D
D. drone si posiziona sopra il rover
E. inquadratura rover con l'ArUco marker
F. calcolo distanza e angolo di rotazione
G. Invio comandi dal computer al rover via bluethoot
H. Rover ruota e avanza
I. fuoco raggiunto?    I1. no > E    I2. si > J
J. Fine: raggiunto obbiettivo

# codice

from ultralytics import YOLO   # Importazione librerie
import cv2
import numpy as np
import mss
import math
import socket


# Collegamento rover

HOST = '192.168.2.3'   # Indirizzo ip del rover
PORT = 65432


# Funzioni per movimento del rover

def rotate(power, gradi):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:   
        s.connect((HOST, PORT))
        comando = f"ROTATE:{power}:{gradi}"
        s.sendall(comando.encode())          # Invio messaggio al server
        data = s.recv(1024)
        print("Rover:", data.decode())


def go_forward(power, gradi):
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))            # Invio messaggio
        comando = f"FORWARD:{power}:{gradi}"
        s.sendall(comando.encode())
        data = s.recv(1024)
        print("Rover:", data.decode())

def stop():
    with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
        s.connect((HOST, PORT))
        comando = "STOP"
        s.sendall(comando.encode())      # Invio messaggio
        data = s.recv(1024)
        print("Rover:", data.decode())



# Caricamento dizionario e delle regole di riconoscimento per il riconoscimento del ArUco

marker_riconoscibili = cv2.aruco.getPredefinedDictionary(cv2.aruco.DICT_4X4_50)    
regole_di_riconoscimento = cv2.aruco.DetectorParameters()
detector = cv2.aruco.ArucoDetector(marker_riconoscibili, regole_di_riconoscimento) # Inizializzazione detector

catturatore_schermo = mss.mss()           # Inizializzazione strumrnto di cattura 
monitor = catturatore_schermo.monitors[1]

angolo_precedente = 0
xc_precedente = 0
yc_precedente = 0
n = 0


model = YOLO("best.pt")        # Caricamento modello yolo addestrato


cv2.namedWindow("Fire Detection - Screen", cv2.WINDOW_NORMAL)  # Creazione di una finestra

while True:
    screenshot = catturatore_schermo.grab(monitor)
    frame = np.array(screenshot)
    frame = cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)    # Cattura schermo e conversione in rgb

    results = model(frame)               # Analisi frame per il riconoscimento del fuoco
    annotated_frame = results[0].plot()

    annotated_frame = cv2.resize(annotated_frame, (800, 600))

    cv2.imshow("Fire Detection - Screen", annotated_frame) # Mostra immagine

    key = cv2.waitKey(1) & 0xFF
    if key == 32:                 # Condizione di uscita
        break

cv2.destroyAllWindows()   # Chiusura finestre


cv2.namedWindow("Guida rover", cv2.WINDOW_NORMAL)   # Creazione di una nuova finestra


while True:
   
        screenshot = catturatore_schermo.grab(monitor)
        frame = np.array(screenshot)                        # Cattura schermo e conversione in gray
        frame = cv2.cvtColor(frame, cv2.COLOR_BGRA2BGR)
        gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
        corners, ids, rejected = detector.detectMarkers(gray)   # Analisi frame per individuare l'ArUco

        if ids is not None:
            print("Marker trovato:", ids.flatten())  
            cv2.aruco.drawDetectedMarkers(frame, corners, ids)    
            marker = corners[0][0]        # Rilevamento delle coordinate dei quattro vertici
            x4, y4 = marker[0]
            x3, y3 = marker[1]
            x2, y2 = marker[2]
            x1, y1 = marker[3]
            dx = x2-x1
            dy = y2-y1
           
            angolo = math.atan2(dy, dx)        # Calcolo dell'angolo di inclinazione
            angolo_deg = angolo*180/math.pi
            print(angolo_deg)
            delta_angolo = angolo_deg - angolo_precedente
            delta_angolo = (delta_angolo + 180) % 360 - 180
           
            xc = (x1 + x2 + x3 + x4) / 4    # Calcolo delle coordinate del centro del Marker
            yc = (y1 + y2 + y3 + y4) / 4
            delta_xc = xc - xc_precedente
            delta_yc = yc - yc_precedente
            xc_frame = frame.shape[1] / 2
            yc_frame = frame.shape[0] / 2 - 20
            distanza_marker = np.sqrt((xc_frame - xc) ** 2 + (yc_frame - yc) ** 2)    # Calcolo distanza tra il centro del frame e il centro del Marker
            lato = np.sqrt((x1 - x2) ** 2 + (y1 - y2) ** 2)
            pixel = 20.5 / lato            # Conversione da pixel a centimetri
            print(distanza_marker * pixel)
            arco = abs(angolo) * 5.9       

            if abs(delta_angolo) <= 5 and abs(delta_xc) <= 5 and abs(delta_yc) <= 5:     # Controllo per vedere se il marker è fermo
                n += 1
            else :
                n = 0

            if (n >= 6 and angolo_deg < 0):               # Roazione rover in senso orario
                rotate(10, (arco / (5.5 * np.pi)) * 360)    # Calcolo angolo rotazione delle ruote per far ruotare il rover di n gradi
                n = 0

            if (n >= 6 and angolo_deg > 0):
                rotate(-10, (arco / (5.5 * np.pi)) * 360)   # Rotazione in senso anteorario
                n = 0
            
                

            
               
            if(abs(angolo_deg) < 7 and n >= 4):    # Movimento in avanti
                go_forward(50, ((distanza_marker * pixel)/(np.pi * 5.5)) * 360)   # Calcolo rotazione ruote per far andare avanti il rover di n centimetri
                
               
           
            angolo_precedente = angolo_deg
            xc_precedente = xc
            yc_precedente = yc
           
           
     
   
        else:
            n = 0
            print("Nessun marker trovato")

        cv2.imshow("Guida rover", frame)      # Mostrare il frame
        cv2.waitKey(1)
