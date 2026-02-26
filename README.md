# Sistema-Avanzado-de-Medicion-de-Reflejos
from machine import Pin  #  Importa la clase Pin para manejar entradas y salidas digitales
import machine  # Permite acceder a registros internos del ESP32
import time  # Maneja los tiempos (sleep)
import random  # Genera numeros aleatorios

#  CONFIGURACIÓN 

PINES = {
    'leds': [12, 13, 14],
    'buzzer': 18,
    'j1': [25, 26, 27, 32],
    'j2': [19, 21, 22, 23],
    'inicio': 16,
    'fin': 4,
    'modo_simon': 17  #Pin para activar modo Simon dice
}

#  REGISTROS 

GPIO_OUT_W1TS_REG = 0x3FF44008  # Dirección del registro para poner un pin en HIGH (3.3V)
GPIO_OUT_W1TC_REG = 0x3FF4400C  # Dirección del registro para poner un pin en LOW (0V)

def encender_pin(pin):  # Función para encender un pin usando registros
    machine.mem32[GPIO_OUT_W1TS_REG] = (1 << pin)  # Activa el bit correspondiente al pin

def apagar_pin(pin):
    machine.mem32[GPIO_OUT_W1TC_REG] = (1 << pin)  # Desactiva el bit correspondiente al pin

# Configurar LEDs y buzzer como salida
for p in PINES['leds']:  # Recorre cada pin de la lista de LEDs
    Pin(p, Pin.OUT)  # Configura cada LED como salida digital

Pin(PINES['buzzer'], Pin.OUT)

#  BOTONES 

BOTON_INICIO = Pin(PINES['inicio'], Pin.IN, Pin.PULL_DOWN)  # Botón inicio como entrada con resistencia interna PULL_DOWN
BOTON_FIN = Pin(PINES['fin'], Pin.IN, Pin.PULL_DOWN)
BOTON_MODO = Pin(PINES['modo_simon'], Pin.IN, Pin.PULL_DOWN)  # Botón para activar Simón

BOTONES_J1 = [Pin(p, Pin.IN, Pin.PULL_DOWN) for p in PINES['j1']]  # Lista de botones jugador 1
BOTONES_J2 = [Pin(p, Pin.IN, Pin.PULL_DOWN) for p in PINES['j2']]  # Lista de botones jugador 2

#  VARIABLES 

puntaje_j1 = 0
puntaje_j2 = 0
modo_juego = 1  # Modo por defecto (1 jugador)
bandera_simon = False  # Variable bandera para activar modo Simón

#  INTERRUPCIÓN SIMÓN 

def activar_simon(pin):  # Función que se ejecuta cuando se presiona botón modo
    global bandera_simon  # Permite modificar variable global
    bandera_simon = True  # Activa modo Simón

BOTON_MODO.irq(trigger=Pin.IRQ_RISING, handler=activar_simon)  # Configura interrupción cuando el botón pasa de apgado a encendido

#  FUNCIÓN ANTI-REBOTE 

def esperar_boton(lista_botones):  # Función que espera hasta que se presione un botón
    while True:
        for i in range(4):  # Recorre los 4 botones
            if lista_botones[i].value():  # Si detecta señal encendida
                time.sleep(0.05)  # Antirebote
                if lista_botones[i].value():  # Verifica nuevamente
                    while lista_botones[i].value():
                        pass
                    return i  # Devuelve índice del botón presionado

#  FUNCIÓN SIMÓN 

def modo_simon():
    global bandera_simon

    print("\n MODO SIMÓN DICE ")

    secuencia = []  # Lista donde se guarda la secuencia generada
    ronda = 0  # Contador de rondas

    while True:

        if BOTON_FIN.value():
            bandera_simon = False
            print("Simón finalizado manualmente")
            return  # Sale del modo Simón

        ronda += 1
        print("Ronda:", ronda)

        nuevo = random.randint(0, 3)  # Genera número aleatorio entre 0 y 3
        secuencia.append(nuevo)  # Lo agrega a la secuencia

        # Mostrar secuencia
        for elemento in secuencia:

            if elemento < 3:  # Si es LED
                encender_pin(PINES['leds'][elemento])
                time.sleep(0.5)
                apagar_pin(PINES['leds'][elemento])
            else:  # Si es buzzer
                encender_pin(PINES['buzzer'])
                time.sleep(0.5)
                apagar_pin(PINES['buzzer'])

            time.sleep(0.3)  # Pausa entre elementos

        # Validar secuencia
        for esperado in secuencia:

            presionado = esperar_boton(BOTONES_J1)

            if presionado != esperado:  # Si se equivoca
                print("Perdiste en ronda", ronda)
                print("Puntaje logrado:", ronda - 1)
                bandera_simon = False
                return

        print("Correcto")
        time.sleep(0.8)

#  MENÚ 

print("Presione INICIO para comenzar")

while not BOTON_INICIO.value():  # Espera hasta que se presione inicio
    pass

print("Seleccione modo:")
print("Botón 1 (J1) → 1 jugador")
print("Botón 2 (J1) → 2 jugadores")

while True:
    if BOTONES_J1[0].value():  # Si presiona botón 1
        modo_juego = 1
        break
    if BOTONES_J1[1].value():  # Si presiona botón 2
        modo_juego = 2
        break

print("Modo seleccionado:", modo_juego)
time.sleep(1)

#  JUEGO NORMAL 

while True:

    if bandera_simon:  # Si se activó Simón
        bandera_simon = False
        modo_simon()
        print("Regresando al modo normal...\n")
        time.sleep(1)

    if BOTON_FIN.value():  # Si presiona finalizar
        break

    time.sleep(random.uniform(1, 10))  # Tiempo aleatorio entre 1 y 10 segundos

    estimulo = random.randint(0, 3)  # Genera estímulo aleatorio
    ganador = None
    inicio_tiempo = time.ticks_ms()  # Guarda tiempo inicial

    error_j1 = False
    error_j2 = False

    # Activar estímulo
    if estimulo < 3:
        encender_pin(PINES['leds'][estimulo])
    else:
        encender_pin(PINES['buzzer'])

    # Esperar respuesta
    while ganador is None:

        if BOTON_FIN.value():
            ganador = "FIN"
            break

        for i in range(4):
            if BOTONES_J1[i].value():
                time.sleep(0.05)
                if BOTONES_J1[i].value():
                    while BOTONES_J1[i].value():
                        pass
                    if i == estimulo:
                        ganador = "Jugador 1"
                        puntaje_j1 += 1
                    else:
                        if not error_j1:
                            puntaje_j1 -= 1
                            error_j1 = True

        if modo_juego == 2:
            for i in range(4):
                if BOTONES_J2[i].value():
                    time.sleep(0.05)
                    if BOTONES_J2[i].value():
                        while BOTONES_J2[i].value():
                            pass
                        if i == estimulo:
                            ganador = "Jugador 2"
                            puntaje_j2 += 1
                        else:
                            if not error_j2:
                                puntaje_j2 -= 1
                                error_j2 = True

    # Apagar estímulo
    if estimulo < 3:
        apagar_pin(PINES['leds'][estimulo])
    else:
        apagar_pin(PINES['buzzer'])

    if ganador == "FIN":
        break

    fin_tiempo = time.ticks_ms()  # Tiempo final
    tiempo_reaccion = time.ticks_diff(fin_tiempo, inicio_tiempo)  # Calcula tiempo de reacción

    print("-------------------------")
    print("Estímulo:", estimulo + 1)
    print("Ganador ronda:", ganador)
    print("Tiempo reacción:", tiempo_reaccion, "ms")
    print("Puntaje J1:", puntaje_j1)

    if modo_juego == 2:
        print("Puntaje J2:", puntaje_j2)

    print("-------------------------")
    time.sleep(1)

print("\n JUEGO FINALIZADO ")
print("Puntaje Final J1:", puntaje_j1)

if modo_juego == 2:
    print("Puntaje Final J2:", puntaje_j2)

