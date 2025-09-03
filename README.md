from machine import Pin, mem32 #controlar GPIO y acceder a registros
from micropython import const #definir vbles constantes (valores que no cambian)
import time, random #genera num aleatorios, para q´ el jugador no sepa que va a aparecer.

# Configuración
led_pins = [2, 4, 5]             # Pines LEDs
buzzer_pin = 18                  # Pin buzzer activo
btn_start = Pin(13, Pin.IN, Pin.PULL_DOWN)  # Botón inicio
btn_modo3 = Pin(14, Pin.IN, Pin.PULL_DOWN)  # Botón modo3
btn_stop  = Pin(27, Pin.IN, Pin.PULL_DOWN)  # Botón stop juego

# Botones jugador 1
btns_j1 = [Pin(15, Pin.IN, Pin.PULL_DOWN), 
           Pin(16, Pin.IN, Pin.PULL_DOWN), 
           Pin(17, Pin.IN, Pin.PULL_DOWN),
           Pin(19, Pin.IN, Pin.PULL_DOWN)]  

# Botones jugador 2
btns_j2 = [Pin(21, Pin.IN, Pin.PULL_DOWN), 
           Pin(22, Pin.IN, Pin.PULL_DOWN), 
           Pin(23, Pin.IN, Pin.PULL_DOWN),
           Pin(25, Pin.IN, Pin.PULL_DOWN)]  

#buzzer activo
buzzer = Pin(buzzer_pin, Pin.OUT)#creacion del buzzer(OUT para q´ mande señales al buzzer)

   #funciones de control para encender y apagar el buzzer
def buzzer_on():                 
    buzzer.value(1)   # Enciende buzzer activo (3,3V)

def buzzer_off():                
    buzzer.value(0)   # Apaga buzzer activo (0V)


# Manejo LEDs con registro 
GPIO_OUT_REG = const(0x3FF44004)   # Registro GPIO de salida

def led_on(pin):
    mem32[GPIO_OUT_REG] |= (1 << pin)   # Pone 1 en la posicion del pin 
                       #| es un OR, pone el 1 sin cambiar los demás

def led_off(pin):
    mem32[GPIO_OUT_REG] &= ~(1 << pin)  # todos 1. excepto en la posion del pin, ahí va 0
                      #& AND con NOT, limpia ese pin en  0 y mantiene los demás
def apagar_todos():
    for p in led_pins: #recorre toda la lista de los pines y los apaga
        led_off(p)
    buzzer_off() #el buzzer también se apaga


# Variables de control 
puntaje = [0, 0]                 # marcador jugadores
modo_juego = 1                   # Modo inicial
juego_activo = True              # Control bucle (si sigue corriendo o termina)


# Funciones
def estimulo_aleatorio():  #no se sabe cual se activa porque es aleatorio      
    return random.randint(0, 3)  # 0-2 LED, 3 buzzer
    #devulve ese num al lugar donde se llamó la función

def antirrebote(boton):   #para evitar que el micro lea varias veces una sola pulsacion       
    time.sleep_ms(50)   #tiempo en que la señal se estabiliza        
    return boton.value() #lee el valor limpio del boton (0-suelto y 1-presionado)
    #devuelve el valor limpio

def esperar_respuesta(objetivo, jugador):  #para esperar la accion del jugador cuando aparece un estimilo
    #objetivo: estímulo que se activo (buzzer o LEDs)
    #jugador: 0=Jugador 1  y  1=Jugador2
    inicio = time.ticks_ms()  #guarda la hora cuando apareció el estimulo   
    while True:  #espera hasta que le jugador presione algo         
        botones = btns_j1 if jugador == 0 else btns_j2 #selecciona los botones que corresponden al jugador
        for i, b in enumerate(botones): #recorre todos los botones de ese jugador (i = índice, b = botón).
            if b.value() == 1:  #si el boton esta presionado           
                if i == objetivo:  #si el botón corresponde al estímulo correcto   
                    fin = time.ticks_ms() #guarda el tiempo en que respondió.
                    return time.ticks_diff(fin, inicio), True #calcula el tiempo de reacción y devuelve el tiempo que fue correcto
                else:          #si fue un boton equivocado            
                    return None, False   #devuelve fallo   

def ronda(jugadores):            
    apagar_todos() #asegura que al inicio no quede ningún LED ni buzzer encendido             
    espera = random.randint(1, 10)  #escoge un tiempo aleatorio entre 1 y 10 segundos
    time.sleep(espera)      # espera ese tiempo antes de mostrar el estímulo    

    salida = estimulo_aleatorio()  #elige LED o buzzer aleatoriamente
    if salida < 3:       #si el valor es 0, 1 o 2 ---enciende un LED        
        led_on(led_pins[salida]) #lo encienda de los LEDs declarados
    else:      #si es 3                  
        buzzer_on()   #enciende el buzzer           

    print("¡Reacciona!")    #muestra un mensaje en consola      
    for j in range(jugadores):    #recorre cada jugador (1 o 2)
        tiempo, correcto = esperar_respuesta(salida, j) #espera la respuesta del jugador
        if correcto:    #si acierta imprime tiempo de reacción y suma un punto          
            print("Jugador", j+1, "tiempo:", tiempo, "ms")
            puntaje[j] += 1       
        else:       #si falla, imprime "falló" y resta un punto              
            print("Jugador", j+1, "falló")
            puntaje[j] -= 1       

    apagar_todos()  #apaga todo para cerrar la ronda         
    time.sleep(1)   #espera 1s antes de la siguiente ronda         

def mostrar_puntajes():     
    #imprimir puntajes      
    print("\nPuntajes:")
    print("Jugador 1:", puntaje[0]) #marcador del J1
    print("Jugador 2:", puntaje[1])  #marcador del J2


# Modo 3 Contrarreloj 
def modo3_contrarreloj():  #se definde la función   
#este modo dura 30s y no por rondas    
    print("\n*** Modo 3: Contrarreloj 30s ***")
    tiempo_inicio = time.ticks_ms()  #guarda el tiempo actual en milisegundos.
    while time.ticks_diff(time.ticks_ms(), tiempo_inicio) < 30000:  #repite el bucle mientras no pasen 30s
        salida = estimulo_aleatorio()  #escoge LED (0,1,2) o buzzer (3).
        if salida < 3:  #si es 0,1,2---se enciende el LED correspondiente.         
            led_on(led_pins[salida])
        else:     #si es 3 enciende el buzzer                      
            buzzer_on()
        inicio = time.ticks_ms()   #guarda la hora en que apareció el estímulo.     

        responded = False     #al inicio, nadie ha respondido          
        while not responded and time.ticks_diff(time.ticks_ms(), inicio) < 3000: 
            #tiempo límite para responder (max 3s) 
            for j in range(2):          
                botones = btns_j1 if j == 0 else btns_j2 #que jugador
                for i, b in enumerate(botones): #selecciona los botones del jugador
                    if b.value() == 1:  #si ese boton esta presionado
                        if i == salida: #si el boton esta presionado, corresponde al estímulo
                            print("Jugador", j+1, "¡acierto!") #imprime acirto y suma 1 punto
                            puntaje[j] += 1
                        else:        #si no coincide, falla y resta un acierto  
                            print("Jugador", j+1, "falló")
                            puntaje[j] -= 1
                        responded = True #sale del ciclo, porque ya hubo respuesta.
                        break #pequeña pausa para evitar sobrecargar 
            time.sleep_ms(20)

        apagar_todos()    #apaga LEDs y buzzer después de cada intento.              
        time.sleep(random.uniform(0.5, 1.5))  #espera un tiempo aleatorio entre 0.5 y 1.5 segundos antes de mostrar el siguiente estímulo.

#cuando se acaben los 30s :
    print("\n*** Fin de contrarreloj ***")
    mostrar_puntajes()
    if puntaje[0] > puntaje[1]:
        print("Ganador: Jugador 1")
    elif puntaje[1] > puntaje[0]:
        print("Ganador: Jugador 2")
    else:
        print("Empate")


# Interrupciones

#se ejecuta cuando hay una interrupción en el modo 3
def activar_modo3(pin):           
    global modo_juego #usa la vble global
    print("\n*** Modo 3 activado ***")
    modo_juego = 3

#se ejecuta cuando se detecta una interrupción en el botón STOP.
def detener_juego(pin):           
    global juego_activo
    print("\n*** Juego detenido por pulsador STOP ***")
    juego_activo = False #rompe el bucle principal, el juego se detiene

#irq: interrupcion en el pin
#trigger=Pin.IRQ_RISING: la interrupción se activa cuando el pin detecta un flanco ascendente (pasa de 0 a 1, es decir, cuando se presiona el botón).
#handler=activar_modo3: función que se ejecuta automáticamente cuando ocurre la interrupción.
btn_modo3.irq(trigger=Pin.IRQ_RISING, handler=activar_modo3)
btn_stop.irq(trigger=Pin.IRQ_RISING, handler=detener_juego)

#Juego principal

#muestra las pciones y guarada el modo
print("Seleccione modo:")         
print("1. Un jugador")
print("2. Dos jugadores")
modo_juego = int(input("Ingrese opción: "))

#El programa se queda detenido hasta que el jugador presione el botón de inicio
print("Presione botón inicio")    
while not btn_start.value():      
    pass

#Dependiendo del modo, ejecuta las rondas normales o el contrarreloj.
print("Juego iniciado!")          
while juego_activo:               
    if modo_juego == 1:           
        ronda(1)
    elif modo_juego == 2:         
        ronda(2)
    elif modo_juego == 3:         
        modo3_contrarreloj()      
        juego_activo = False   
    #Se muestran los puntajes y se pregunta si quieren otra ronda; si no, el juego se detiene.   
    if juego_activo:              
        mostrar_puntajes()        
    if modo_juego in [1, 2] and juego_activo:  
        cont = input("¿Otra ronda? (s/n): ")
        if cont.lower() != "s":   
            juego_activo = False

#Al salir del bucle principal, se muestran puntajes finales y se declara al ganador o empate
print("\nJuego terminado")        
mostrar_puntajes()                
if puntaje[0] > puntaje[1]:
    print("Ganador: Jugador 1")
elif puntaje[1] > puntaje[0]:
    print("Ganador: Jugador 2")
else:
    print("Empate")
