# Reto 5 Sistemas IoT- Capa de Aplicación - Lógica

## Modificaciones de código - capa de aplicación

Luego de preparar la infraestructura y validar la correcta ejecución de los servicios en cada una de sus instancias:

<image src="images/reto5-aws-instancias2.png" width="100%" height="100%" alt="instancias-aws">

Instancia broker MQTT:

<image src="images/reto5-mqtt-log-2.png" width="90%" height="90%" alt="instancias-aws">

Base de datos TimeScale:

<image src="images/reto5-timescale-01.png" width="100%" height="100%" alt="alert2">

Aplicación Viewer:

<image src="images/reto5-viewer-mapa-historico-temp.png" width="100%" height="100%" alt="instancias-aws">

El equipo procede a analizar el código para entender donde se deben realizar las modificaciones necesarias para adicionar el comportamiento de un nuevo evento, encontrando como alternativa para la implementación en la capa de aplicación, especificamente en la función `analyze_data()` del componente `control/monitor`, donde se consultan los datos de la última hora, agrupados por estación y variable de monitoreo, para luego compararlos con los rangos configurados de la variable y, en caso de no estar dentro del rango, enviar un mensaje de alerta al broker MQTT.

Para adicionar el comportamiento de un nuevo evento, el equipo decide adicionar a la validación actual de rangos, una validación preliminar cuando la variable se esté acercando al limite superior `Item["check_value"] > (min_value + (max_value-min_value)*0.8)`, para generar una alarma preventiva previa a la alarma definitiva. Para facilitar los cambios en la capa de dispositivos, se reutiliza el tópico country/state/city/user/in, pero se cambia el prefijo del mensaje de `ALERT` a `RETO`:

```
    alerts = 0
    for item in aggregation:
        alert = False
        alertReto = False
        ...
        if item["check_value"] > max_value or item["check_value"] < min_value:
            alert = True
        elif item["check_value"] > (min_value + (max_value-min_value)*0.8):
            alertReto = True

        if alert or alertReto:
            if alert:
                message = "ALERT {} {} {}".format(variable, min_value, max_value)
            else:
                message = "RETO {} {} {}".format(variable, min_value, max_value)
            topic = '{}/{}/{}/{}/in'.format(country, state, city, user)
            print(datetime.now(), "Sending alert to {} {} {}".format(topic, variable, message))
            client.publish(topic, message)
```

Adicionalmente, se ajusta la ejecución de la función en el cron de 5 a 2 minutos 
```
def start_cron():
    ...
    schedule.every(2).minutes.do(analyze_data)
    print("Servicio de control iniciado")
    while 1:
        schedule.run_pending()
        time.sleep(1)
```

## Modificaciones de código - capa de dispositivos

Con respecto a la capa de dispositivo, incialmente el equipo se planteó presentar la información de la nueva alerta en la pantalla, pero dado el estado de la misma, se decide encender un led cuando llegue la nueva alerta con prefijo `RETO`:
```
void receivedCallback(char* topic, byte* payload, unsigned int length) {
  Serial.print("Received [");
  Serial.print(topic);
  Serial.print("]: ");
  String data = "";
  for (int i = 0; i < length; i++) {
    data += String((char)payload[i]);
  }
  Serial.print(data);
  if (data.indexOf("ALERT") >= 0) {
    alert = data;
  }else if(data.indexOf("RETO") >= 0){
    actuator = true;
  }
}
```
...
```
void measure() {
  if ((millis() - measureTime) >= MEASURE_INTERVAL * 1000 ) {
    Serial.println("\nMidiendo variables...");
    measureTime = millis();
    
    temp = readTemperatura();
    humi = readHumedad();

    // Se chequea si los valores son correctos
    if (checkMeasures(temp, humi)) {
      // Se envían los datos
      sendSensorData(temp, humi); 
    }

    if(actuator && increment<=10){
      increment++;
      state = !state;
      digitalWrite(ACTUATOR_1_PIN, state);
      Serial.println("\nActuador activo...");
    }else if(increment>10){
      increment = 0;
      actuator=false;
      digitalWrite(ACTUATOR_1_PIN, false);
    }
  }
}
```

## Ejecución de Pruebas

Para ejecutar las pruebas, se redespliegan los cambios en el componente de alertas en la instancia `IOT System/IoT Alert App`. Inicialmente las variables se mantienen en rango, sin generar alertas, por lo que se procede a afectar el ambiente y en la tercera revisión por parte del componente `control/monitor` se genera la primera alerta de humedad por rangos: 

<image src="images/reto5-alert-11.png" width="100%" height="100%" alt="alert1">


Para continuar la prueba, se ajustan los rangos mínimo y máximo de la humedad para forzar la nueva lógica implementada: 

<image src="images/reto5-viewer-variables.png" width="90%" height="90%" alt="variables-rangos">

Inicialmente se continuan generando alertas por rango para la variable temperatura, y en la cuarta revisión, finalmente se activa la lógica del nuevo evento, `51.72 > 49+(3*0.8) => 51.72 > 51.4`, enviando la nueva alerta con el sufijo `RETO`:

<image src="images/reto5-alert-13.png" width="100%" height="100%" alt="alert2">

Del lado de la capa de dispositivos, se evidencia la nueva alerta en el log:

<image src="images/reto5-capa-dispositivo-consola-reto-humedad.jpg" width="72%" height="72%" alt="alert-device">

y en el comportamiento del actuador implementado, un led para  nuestro caso:

<video src="images/video-actuador.mp4" controls="controls" width="45%" height="45%"> </video>