# Documentación de la Aplicación Global-Sanctions-Consolidator

## 1. Introducción

La aplicación *Global-Sanctions-Consolidator* está diseñada para automatizar la verificación de actualizaciones de datos en las listas de sanciones de la Oficina de Control de Activos Extranjeros (OFAC), la Organización de las Naciones Unidas (ONU), la Unión Europea (UE) y la Oficina del Superintendente de Instituciones Financieras de Canadá (OSFI). Su objetivo principal es reducir el tiempo necesario para este proceso, eliminar el error humano y permitir la consolidación de registros que no existan en la base de datos pero sí se encuentren en estas listas de sanciones internacionales. Esta aplicación es intuitiva, fácil de usar y eficiente en sus respuestas, generando archivos útiles para el área de sanciones.

---

## 2. Funcionamiento Interno

### 2.1. Inicio y Carga de Recursos

1. La aplicación inicializa la interfaz gráfica mediante **PyQt6**.
2. Se configuran los botones principales:
    - **OFAC**  
        - **Generar Archivos de Actualización**: Descarga y procesa los datos más recientes de reportes realizados.  
        - **Generar Lista Completa**: Descarga y procesa todas las entidades sancionadas activamente.  

    - **UE**  
        - **Generar Archivo de Actualización**: Descarga y procesa los datos más recientes de reportes realizados.  
        - **Selector de Fecha de Último Corte**: Permite seleccionar la fecha del último corte de datos a procesar.  

    - **ONU**  
        - **Generar Archivo de Actualización**: Descarga y procesa los datos más recientes de reportes realizados.  

    - **OSFI**  
        - **Generar Archivo de Actualización**: Descarga y procesa los datos más recientes de reportes realizados.

---

### 2.2. Actualización de Lista

---

### 2.2.1. OFAC

#### 2.2.1.1 Descarga de Datos de Sanciones

1. La aplicación se conecta a la API de OFAC:
    - `https://sanctionslistservice.ofac.treas.gov/changes/latest` para obtener la última actualización realizada a la lista, donde entran modificaciones, eliminaciones y agregaciones.
2. Los datos se descargan en formato **XML**.
3. Se extraen los siguientes datos:
    - **ID de OFAC** (cadena de texto).
    - **Nombre principal en formato latino** (cadena de texto).
    - **Listado de alias asociados** (lista de texto).
    - **Listado de documentos asociados** (lista de texto).
    - **Tipo de acción** (cadena de texto).
    - **Fecha de publicación de la actualización** (cadena de texto).
4. Si el tipo de acción no está especificado, el tipo de acción es **modificación**. En estos casos, OFAC **no proporciona información del nombre de la entidad**, por lo que la aplicación busca en `Transfer.xlsx` utilizando el **ID de OFAC** para obtener el nombre si está registrado.

#### 2.2.1.2. Generación del Archivo OFAC

Con la información descargada, se guarda el primer archivo llamado **`OFAC_{fecha de actualización}.xlsx`**, donde se almacenan los datos sin tratamiento adicional.

#### 2.2.1.3. Procesamiento de Alias

1. Para cada entidad sancionada, se analiza el listado de alias para detectar posibles alias similares al nombre real. En caso de coincidencia significativa, el alias se considera un nuevo registro debido al riesgo de exclusión.
2. Se usa un **algoritmo propio** que:
    - Separa cada alias en palabras individuales.
    - Evalúa la similitud de cada palabra del alias con el nombre original de la entidad.
    - Calcula un promedio de coincidencia con cada palabra del alias y el nombre real.
3. Se descarta un alias si:
    - Tiene menos de **10 caracteres**.
    - Contiene **números**.
    - No supera un umbral de similitud predefinido.
4. Los alias aceptados se conservan en la columna de alias, mientras que los que no cumplen los requisitos son eliminados del listado.

#### 2.2.1.4. Tratamiento de Identificaciones

Los documentos de identificación se procesan de la siguiente manera:

1. Si el documento **no** es:
    - **Cédula de Ciudadanía de Colombia**.
    - **NIT de Colombia**.
    - **Pasaporte compuesto solo por caracteres numéricos**.
      Entonces, se eliminan las identificaciones originales y se reemplazan con una lista que contiene únicamente el **ID de OFAC**.
2. Si el documento cumple con los criterios anteriores, se almacena en el listado. Si hay múltiples documentos válidos, se conservan todos.

#### 2.2.1.5. Tabla de Combinaciones

Se genera una tabla intermedia para el uso de combinaciones de nombres y documentos.

1. Cada entidad tiene:
    - Un **nombre**.
    - Un **listado de alias**.
    - Un **listado de documentos**.
2. Se crea una tabla donde cada posible combinación de nombre y documento se convierte en un registro único.

Ejemplo:

| Nombre | Alias       | Documento              |
| ------ | ----------- | ---------------------- |
| Juan   | ['Juanito'] | ['CC 123', 'PAS 1234'] |

Se transforma en:

| Nombre  | Documento |
| ------- | --------- |
| Juan    | CC 123    |
| Juan    | PAS 1234  |
| Juanito | CC 123    |
| Juanito | PAS 1234  |

#### 2.2.1.6. Comparación con Transferencias

1. Se compara cada nombre con todos los nombres en **`Transfer.xlsx`**.
2. Se determina el nombre más similar y se trae, con esto se logra mostar el nombre similar, su respectivo **ID de OFAC** y el porcentaje de similitud.
3. Se genera el archivo final **`OFAC_Transfer_{fecha de actualización}.xlsx`**.

---

#### 2.2.2. UE

#### 2.2.2.1 Descarga de Datos de Sanciones

1. La aplicación descarga el archivo de la UE:
    - `https://webgate.ec.europa.eu/fsd/fsf/public/files/csvFullSanctionsList_1_1/content?token=dG9rZW4tMjAxNw` para obtener la lista completa de sancionados.
2. Los datos se descargan en formato **CSV**.
3. Se extraen los siguientes datos:  
    - **Identification Number** (cadena de texto).  
    - **NameAlias Whole Name** (cadena de texto).  
    - **NameAlias Name Language** (cadena de texto).  
    - **Entity Regulation Publication Date** (cadena de texto).  

#### 2.2.1.2. Aplicar Filtros

- Se convierte la columna **Entity Regulation Publication Date** a formato de fecha.  
- Se filtran los registros cuya **Entity Regulation Publication Date** sea mayor o igual a la fecha indicada en el selector de fechas.  
- Se reemplazan los valores nulos en la columna **NameAlias Name Language** con una cadena de texto vacía.  
- Se filtran los registros donde **NameAlias Name Language** sea **"EN"**, **"ES"** o vacío.  
- Se eliminan los registros donde **NameAlias Whole Name** sea nulo.  
- Se seleccionan únicamente las columnas **Identification Number** y **NameAlias Whole Name**, renombrándolas como **ID** y **NOMBRE**, respectivamente.  

#### 2.2.1.3. Generación del Archivo UE

Con la información filtrada, se guarda el primer archivo llamado **`UE_{fecha de actualización}.xlsx`**, donde se almacenan los datos sin la comparación con el **`Transfer.xlsx`**.

#### 2.2.1.4. Comparación con Transferencias

1. Se compara cada nombre con todos los nombres en **`Transfer.xlsx`**.
2. Se determina el nombre más similar y se trae, con esto se logra mostar el nombre similar y el porcentaje de similitud.
3. Se genera el archivo final **`UE_Transfer_{fecha de actualización}.xlsx`**.

---

### 2.3. Descarga de Entidades

---

### 2.3.1. OFAC

#### 2.3.1.1. Descarga de Datos de Entidades

1. La aplicación se conecta a la API de OFAC:
    - `https://sanctionslistservice.ofac.treas.gov/entities` para obtener la lista completa de entidades sancionadas activamente.
2. Los datos se descargan en formato **XML**.
3. Se extraen los siguientes datos:
    - **ID de OFAC** (cadena de texto).
    - **Nombre principal en formato latino** (cadena de texto).
    - **Listado de alias asociados** (lista de texto).
    - **Listado de documentos asociados** (lista de texto).
    - **Fecha de publicación de la actualización** (cadena de texto).
4. A diferencia del proceso de actualización de lista, en esta descarga:
    - **No hay tipo de acción**.
    - **No se consulta el `Transfer.xlsx`**.
    - Los datos extraídos se guardan directamente en un archivo de salida.

5. Se genera el archivo **`OFAC_Entity_{pub_date}.xlsx`**, donde `{pub_date}` representa la fecha de publicación de los datos descargados.

A pesar de ser un proceso más directo y sin intervención de algoritmos, se espera que su ejecución tome significativamente más tiempo que el proceso de actualización, debido al peso de la base de datos y la velocidad de descarga de todas las entidades.

---

#### 2.4. Finalización y Resultados

1. Si el proceso finaliza correctamente, la interfaz muestra un **mensaje de éxito**.
2. Si ocurre un error, la interfaz cambia a **color rojo** y muestra una **X**, indicando la razón del fallo.
3. Los archivos generados se almacenan en la carpeta `output_folder`, donde pueden **ser modificados, eliminados o extraídos sin problema**. Sin embargo, **no deben abrirse hasta que el proceso termine**.

# 3. Manual de Uso

## 3.1. Instalación y Requisitos Previos

Debido a que la aplicación incluye una carpeta `python-3`, no es necesario instalar librerías ni el intérprete de Python. Sin embargo, en caso de necesitarlo, los requisitos son los siguientes:

-   Tener **Python 3.12 o superior** instalado.
-   Instalar las dependencias ejecutando:

    ```sh
    pip install -r requirements.txt
    ```

## 3.2. Ejecución de la Aplicación

### En Windows

Hacer doble clic en `run_win.bat`.

## 3.3. Uso de la Aplicación

1. **Abrir la aplicación**.
2. Presionar el botón con la acción requerida. A continuación, algunos factores a tener en cuenta:  
   - Para el caso de **UE**, es necesario seleccionar la fecha de **último corte**. De lo contrario, el sistema tomará la fecha actual como el último corte, lo que podría generar un archivo vacío.  
3. Cuando un proceso finaliza con éxito, se muestra un **mensaje de confirmación**.
4. Si ocurre un error:
    - La interfaz cambia a **color rojo**.
    - Se muestra una **X** indicando el problema.
    - Las razones comunes incluyen:
        - Falta de conexión a Internet.
        - `Transfer.xlsx` o `output_folder` no existen.
        - Cambios en la estructura de los datos de la OFAC.

## 3.4. Errores Comunes y Solución

-   **Error de conexión**: Verifique su conexión a Internet y vuelva a intentarlo.
-   **Archivo `Transfer.xlsx` no encontrado**: Asegúrese de que el archivo esté en la ubicación correcta, dentro de la carpeta principal, donde también debería encontrarse la carpeta `app`.
-   **`output_folder` no existe**: Cree manualmente la carpeta en la ubicación adecuada.
-   Cualquier error en la aplicación puede evitarse fácilmente si no se modifica la ubicación de los documentos, ya que esto puede provocar fallos al no encontrar los recursos necesarios. Además, es fundamental seguir la estructura de nombres en tablas, columnas y archivos para garantizar el correcto funcionamiento del sistema.  

---

## 4. Conclusión

Luego de realizar las validaciones correspondientes con el área de sanciones, se confirmó que la información generada por la aplicación es precisa y satisfactoria. Como resultado, la herramienta ahora puede ser utilizada por el personal de sanciones, quienes recibirán capacitación para su correcto uso. Esto permitirá reducir la carga de trabajo y mejorar significativamente el tiempo de respuesta ante los cambios inesperados que surgen en cada actualización de las listas de OFAC, ONU, UE y OSFI.