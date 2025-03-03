# Workflow de Procesamiento de Datos de Entregas en n8n

Este workflow automatiza el procesamiento y la transformación de datos provenientes de una hoja de cálculo de Google Sheets. Se encarga de limpiar y formatear la información, filtrar registros según el estado de la entrega, registrar la data procesada en otra hoja y enviar notificaciones por correo electrónico a los clientes. Aunque originalmente se implementó en Python, en este caso se realizó utilizando n8n, aprovechando sus nodos nativos y la posibilidad de incluir código JavaScript para personalizar la transformación de datos.

## Tabla de Contenidos

- [Requisitos](#requisitos)
- [Estructura del Workflow](#estructura-del-workflow)
  - [1. Trigger de Google Sheets](#1-trigger-de-google-sheets)
  - [2. Transformación de Datos](#2-transformación-de-datos)
    - [a. Formateo de Fechas](#a-formateo-de-fechas)
    - [b. Limpieza de Espacios en "Cliente"](#b-limpieza-de-espacios-en-cliente)
    - [c. Conversión de Valor (Comas por Puntos)](#c-conversión-de-valor-comas-por-puntos)
  - [3. Filtrado de Registros](#3-filtrado-de-registros)
  - [4. Registro en Google Sheets](#4-registro-en-google-sheets)
  - [5. Generación de Resumen](#5-generación-de-resumen)
  - [6. Notificaciones vía Correo Electrónico](#6-notificaciones-vía-correo-electrónico)
  - [7. Conversión del Resumen a Archivo](#7-conversión-del-resumen-a-archivo)
- [Conexiones y Flujo de Datos](#conexiones-y-flujo-de-datos)
- [Configuración y Credenciales](#configuración-y-credenciales)
  - [Credenciales de Google](#credenciales-de-google)
- [Cómo Ejecutar el Workflow](#cómo-ejecutar-el-workflow)
- [Conclusiones](#conclusiones)

## Requisitos

- **n8n:** Instancia de n8n correctamente configurada y en ejecución.
- **Google Sheets:** Acceso a las hojas de cálculo utilizadas, con los siguientes IDs:
  - Hoja de origen (por ejemplo, “Entregas_Pendientes”).
  - Hoja destino para registrar las entregas procesadas.
- **Gmail:** Cuenta configurada y credenciales para el envío de correos electrónicos.
- **Conocimientos Básicos:** Familiaridad con n8n, Google Sheets y el uso de código JavaScript para transformaciones personalizadas.

## Estructura del Workflow

### 1. Trigger de Google Sheets

- **Nodo:** *Google Sheets Trigger*
- **Función:**  
  - Monitorea una hoja de cálculo en Google Sheets para detectar la adición de nuevas filas.
  - Se activa al detectar el evento `rowAdded` en la hoja de cálculo.

### 2. Transformación de Datos

Esta sección se encarga de limpiar y estandarizar la información proveniente de la hoja de origen.

#### a. Formateo de Fechas

- **Nodo:** *Formateando fechas* (Código)
- **Función:**  
  - Recibe la fecha del pedido en diversos formatos (por ejemplo, `YYYY/MM/DD`, `DD-MM-YYYY` o formatos con nombres de meses).
  - Detecta el separador utilizado ("/", "-" o espacios) y transforma la fecha al formato estándar `YYYY-MM-DD`.
- **Beneficio:**  
  - Garantiza la consistencia en el formato de fecha para futuras operaciones y registros.

#### b. Limpieza de Espacios en "Cliente"

- **Nodo:** *Limpieza espacios vacios* (Código)
- **Función:**  
  - Elimina espacios innecesarios al inicio y final del campo "Cliente".
  - Sustituye múltiples espacios internos por un solo espacio.
- **Beneficio:**  
  - Evita errores de procesamiento debidos a espacios redundantes en los nombres de clientes.

#### c. Conversión de Valor (Comas por Puntos)

- **Nodo:** *Comas por puntos en Valor* (Código)
- **Función:**  
  - Convierte el campo "Valor" a cadena, elimina espacios, reemplaza comas por puntos y lo parsea a número.
  - Formatea el valor a dos decimales.
- **Beneficio:**  
  - Asegura la correcta interpretación y formato de valores monetarios, facilitando cálculos posteriores.

### 3. Filtrado de Registros

- **Nodo:** *Filter*
- **Función:**  
  - Filtra aquellos registros cuyo campo `Estado_Entrega` sea diferente a “Devuelto”.
- **Beneficio:**  
  - Se evita el procesamiento de entregas devueltas, asegurando que solo se manipulen los registros pertinentes.

### 4. Registro en Google Sheets

- **Nodo:** *Google Sheets* (Operación: Append)
- **Función:**  
  - Registra los datos procesados en una hoja de cálculo destino.
  - Mapea columnas específicas: `ID_Entrega`, `Fecha_Pedido`, `Cliente`, `Correo_Cliente`, `Ciudad`, `Estado_Entrega` y `Valor`.
- **Beneficio:**  
  - Centraliza la información procesada para futuras consultas y auditorías.

### 5. Generación de Resumen

- **Nodo:** *Code* (Resumen)
- **Función:**  
  - Procesa todos los registros para calcular:
    - El número total de entregas procesadas.
    - Un resumen por ciudad de las entregas pendientes.
    - El monto total acumulado de las entregas marcadas como “Entregado”.
  - Genera un resumen en formato de texto que incluye:
    - **Total de entregas procesadas**
    - **Ciudades con entregas pendientes** (listado de cada ciudad y su cantidad)
    - **Monto total de entregas realizadas** (formateado a dos decimales)
- **Beneficio:**  
  - Ofrece una visión global del rendimiento y estado de las entregas.

### 6. Notificaciones vía Correo Electrónico

- **Nodo:** *If*  
  - **Función:**  
    - Evalúa el campo `Estado_Entrega` para determinar el mensaje a enviar.
    - Si el estado es “Pendiente”, se envía un correo notificando que el pedido está en camino.
    - Si el estado es “Entregado”, se envía un correo confirmando la entrega exitosa.
- **Nodos de Correo:**  
  - *Gmail* para pedidos en camino.
  - *Gmail1* para pedidos entregados.
- **Beneficio:**  
  - Mejora la comunicación con el cliente, proporcionando información actualizada sobre el estado de su pedido.

### 7. Conversión del Resumen a Archivo

- **Nodo:** *Convert to File*
- **Función:**  
  - Convierte el resumen generado en un archivo de texto llamado “reporte_resumen”.
- **Beneficio:**  
  - Permite almacenar o distribuir el resumen del proceso en un formato fácilmente accesible y legible.

## Conexiones y Flujo de Datos

El flujo del workflow se organiza de la siguiente manera:

1. **Inicio:**  
   - El nodo *Google Sheets Trigger* inicia el proceso al detectar un nuevo registro.

2. **Procesamiento Secuencial:**  
   - El registro pasa por los nodos de transformación: formateo de fechas, limpieza de espacios y conversión de valor.

3. **Filtrado:**  
   - Se filtran los registros que no corresponden a entregas devueltas.

4. **Registro y Resumen:**  
   - Los datos se registran en una hoja de destino y, de forma paralela, se genera un resumen del proceso.

5. **Notificaciones:**  
   - Dependiendo del estado de la entrega, se envía el correo correspondiente.

6. **Finalización:**  
   - Se convierte el resumen en un archivo para su uso posterior.

## Configuración y Credenciales

### Credenciales de Google

Para integrar correctamente Google Sheets y Gmail en el workflow, es necesario configurar las credenciales de Google en n8n. Sigue estos pasos:

1. **Acceder a la documentación:**  
   - Revisa la guía oficial de n8n sobre [Credenciales de Google](https://docs.n8n.io/integrations/builtin/credentials/google/) para obtener instrucciones detalladas.

2. **Crear un Proyecto en Google Cloud Platform:**  
   - Accede a [Google Cloud Console](https://console.cloud.google.com/).
   - Crea un nuevo proyecto o selecciona uno existente.
   - Habilita las APIs necesarias (por ejemplo, Google Sheets API y Gmail API).

3. **Configurar OAuth2:**  
   - En la sección de APIs y Servicios, crea unas credenciales OAuth2.
   - Registra la URI de redirección proporcionada por n8n.
   - Copia el Client ID y el Client Secret.

4. **Configurar en n8n:**  
   - En la interfaz de n8n, ve a la sección de credenciales.
   - Agrega nuevas credenciales para Google Sheets Trigger y Google Sheets (o Gmail) utilizando el Client ID y el Client Secret obtenidos.
   - Sigue la guía de n8n para completar la configuración y autorizar el acceso a tu cuenta de Google.

### Otros Aspectos de Configuración

- **Google Sheets:**  
  - Configura dos conexiones:
    - Una para el *Trigger* (hoja de origen).
    - Otra para el nodo de *Append* (hoja destino).
  - Los identificadores de documento y de hoja se especifican en los parámetros de cada nodo.

- **Gmail:**  
  - Se requiere la configuración de credenciales OAuth2 para los nodos de envío de correo (*Gmail* y *Gmail1*).

- **n8n:**  
  - Asegúrate de que la instancia de n8n tenga acceso a Internet para comunicarse con las APIs de Google y Gmail.
  - Verifica los parámetros de ejecución y la secuencia de nodos para confirmar que el flujo de datos es correcto.

## Cómo Ejecutar el Workflow

1. **Importación:**  
   - Importa el archivo `My_workflow.json` en tu instancia de n8n.

2. **Configuración:**  
   - Actualiza las credenciales y los IDs de las hojas de cálculo según tus necesidades.
   - Verifica la configuración de cada nodo para adaptarlo a tu entorno (por ejemplo, nombres de campos y estructura de la data).

3. **Activación:**  
   - Activa el workflow en n8n.
   - Realiza pruebas agregando nuevos registros a la hoja de origen y verifica que:
     - Los datos se transformen correctamente.
     - Se almacenen en la hoja de destino.
     - Se genere el resumen y se envíen los correos electrónicos correspondientes.

## Conclusiones

Este workflow en n8n demuestra cómo automatizar el procesamiento de datos integrando Google Sheets y Gmail. Gracias a la combinación de nodos predefinidos y la flexibilidad del código JavaScript, se logra:

- **Estandarización de Datos:**  
  - Mediante el formateo de fechas y la limpieza de campos.
- **Filtrado Efectivo:**  
  - Procesando únicamente los registros pertinentes y evitando entradas no deseadas.
- **Comunicación Automatizada:**  
  - Notificando a los clientes en función del estado de sus pedidos.
- **Generación de Reportes:**  
  - Creando un resumen integral que facilita el análisis y seguimiento del proceso.



## Contacto

- **Autor**: [Brian Journeyt Rodriguez Vivas ](brianjourneytrodriguezvivas@gmial.com)
- **Repositorio**: [[https://github.com/brianrodriguezvivas/plaza_medicina]([https://github.com/brianrodriguezvivas/plaza_medicina_n8n)
La estructura modular del workflow permite ampliar o modificar secciones del proceso de forma sencilla, integrando nuevas fuentes de datos o servicios sin necesidad de rehacer el proceso desde cero.
