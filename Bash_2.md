
# Uso basico del lenguaje Bash
> **TEMAS SELECTOS: INTRODUCCIÓN AL ANÁLISIS DE SECUENCIACIÓN MASIVA**  
> Gustavo Ibrahim Giles Pérez
---
## Temas
* Inspección y Manipulación de Archivos NGS (`grep`, `cut`, `sort`, `uniq`)
* Comandos de redireccion y sustitucion (`|`, `>`, `>>`, `awk`, `sed`)
* *Set variables*, *for loop* y procesamiento por lotes de archivos FASTQ
* Scripts ejecutables (`.sh`)
* Buenas practicas bioinformaticas
---

## Inspección y manipulación de archivos NGS

### **1.1. Inspección Rápida de Datos de Secuenciación**

Los archivos **FASTQ** y **FASTA** de secuenciación masiva suelen ser pesados. No debemos abrirlos con editores de texto.
* **Comando `head` y `tail`**: Permiten visualizar el inicio o el final de un archivo.
* **Comando `less`**: Permite desplegar archivos de forma interactiva sin cargarlos completamente en memoria RAM.
* **Archivos comprimidos (`.gz`)**: Usualmente las lecturas NGS vienen comprimidas. Usamos **`zcat`** o **`zless`** para inspeccionarlas sin descomprimir.

```bash
# Visualizar las primeras 8 líneas (2 lecturas FASTQ)
head -n 8 amphipoda_genomes.fastq

# Inspeccionar un archivo FASTQ comprimido (.fastq.gz)
zcat file_1.fastq.gz | head -n 4
```
### **1.2. Buscando patrones con grep**
El comando `grep` es la herramienta fundamental para buscar secuencias, identificadores o adaptadores.

- Busqueda exacta: Encontrar secuencias especificas o patrones de calidad.
- Conteo de coincidencias (-c): Contar cuantas veces aparece un patron.
- Inversion de coincidencia (-v): Excluir lineas que contengan un patron.

Ejemplos:
```bash
# Contar el numero total de lecturas en un archivo FASTQ (buscando el identificador '@'):
grep -c "^@" muestra_R1.fastq

# Buscar la presencia de un adaptador de secuenciacion especifico:
grep "AGATCGGAAGAG" muestra_R1.fastq
```
![alt text](https://github.com/TavoIbrahim/AnalisisDatosGenomicos/blob/8789a9a77b671740e91b5a7512d21999d8a5c151/RegularExp.gif)

### **1.3. Ordenamiento y filtrado con cut, sort y uniq**
- Comando `cut`: Extrae columnas especificas delimitadas por caracteres (por ejemplo, tabuladores en archivos SAM/BED).
- Comando `sort`: Ordena lineas de texto alfabetica o numericamente (-n para numeros, -k para especificar columna).
- Comando `uniq`: Elimina o cuenta lineas duplicadas consecutivas (siempre debe usarse tras sort).

Ejemplo:
```bash
# Extraer la columna de nombres de cromosomas (columna 1) de un archivo BED:
cut -f 1 anotaciones.bed | sort | uniq -c
```
--------------------------------------------------------------------------------
### Ejercicio 1

* Usa el comando zcat y grep para contar cuantas lecturas en muestra1.fastq.gz contienen la secuencia TAAGGTAGCCAAATGCCTCGTCATCTAATTAGTGACGCGCATGAATGGAT.
* Calcula el numero total de lecturas dividiendo el numero total de lineas del archivo entre 4 usando wc -l.
--------------------------------------------------------------------------------

## Comandos de redireccion y sustitucion (`|`, `>`, `>>`, `awk`, `sed`)

### 2.1. Redireccion de entrada y salida
- Operador > (Sobrescribir): Envia la salida de un comando a un archivo nuevo (si el archivo existe, lo reemplaza).
- Operador >> (Anadir): Agrega la salida al final de un archivo existente sin borrar su contenido.

Ejemplos:
```bash
# Guardar las secuencias que contienen un motivo en un archivo nuevo:
grep "AAGCTT" muestra_R1.fastq > lecturas_con_motivo.txt

# Registrar una marca de tiempo en un archivo de log:
echo "Procesamiento finalizado a las $(date)" >> reporte.log
```

### 2.2. Pipes (|)
Un pipe (pipe |) conecta la salida estandar de un comando directamente con la entrada del siguiente comando, evitando crear archivos intermedios innecesarios en el disco duro.

Ejemplo:
```bash
# Extraer solo las secuencias (linea 2 de cada bloque de 4) de un FASTQ y contar las mas frecuentes:
zcat muestra_R1.fastq.gz | paste - - - - | cut -f 2 | sort | uniq -c | sort -nr | head -n 5
```

### 2.3.  awk y sed
- Comando sed (Stream Editor): Ideal para buscar y reemplazar texto.
- Comando awk: Un lenguaje completo para procesar datos estructurados en columnas (como archivos SAM, VCF, GTF o FASTQ).

Ejemplos:
```bash
# Cambiar el formato de cabecera de un archivo FASTA usando sed:
sed 's/>chr/>Cromosoma_/' secuencias.fasta > secuencias_renombradas.fasta

# Filtrar un archivo BED dejando solo regiones mayores a 1000 pares de bases usando awk:
awk '($3 - $2) > 1000 {print $0}' regiones.bed
```
--------------------------------------------------------------------------------
### Ejercicio 2

Tienes un archivo de variantes llamadas en formato VCF (variantes.vcf). Construye una pipe de una sola linea que:
* Filtre las lineas que no son comentarios (que no empiezan por #).
* Filtre aquellas variantes con una calidad de mapeo (columna 6) superior a 30.
* Guarda el resultado en variantes_alta_calidad.vcf.
--------------------------------------------------------------------------------

## 3. *Set variables*, *for loop* y procesamiento por lotes de archivos FASTQ

### 3.1. Uso de variables en bash
Las variables permiten almacenar rutas, nombres de muestras y parametros para reutilizarlos.

- Declaracion: Sintaxis estricta sin espacios alrededor del signo '='.
- Expansion: Se invoca el valor anteponiendo el simbolo '$'.

Ejemplo:
```bash
DIRECTORIO_ENTRADA="/datos/ngs/raw_fastq"
MUESTRA="Paciente01_R1"

echo "Procesando la muestra $MUESTRA localizada en$DIRECTORIO_ENTRADA"
```

### 3.2. Sustitucion de nombres y loops (basename y expansion)
Para procesar datos en cadena, es critico necesario manipular extensiones de archivo automaticamente.

Ejemplo:
```bash
FILE="muestras/NGS_Sample01.fastq.gz"

# Extraer el nombre del archivo sin la ruta:
NAME=$(basename "$FILE")

# Remover extension especifica usando expansion de variables:
SAMPLE_ID=$(basename "$FILE" .fastq.gz)
```

### 3.3. Automatizacion con loops `for`
Un loop realiza la misma secuencia de comandos para una lista de archivos automaticamente.

Ejemplo:
```bash
for ARCHIVO in *.fastq.gz
do
    BASE=$(basename "$ARCHIVO" .fastq.gz)
    echo "Iniciando analisis de calidad para: $BASE"
    # fastqc "$ARCHIVO" --outdir=out/
done
```

--------------------------------------------------------------------------------
### Ejercicio 3

Elabora un loop que explore todos los archivos con extension .fastq en tu wd y que por cada archivo, imprima en la terminal un mensaje con la siguiente estructura:
```bash
"La muestra [NOMBRE_MUESTRA] tiene [NUMERO] lineas en total"
```
--------------------------------------------------------------------------------

## 4. Scripts ejecutables

### 4.1. Anatomia de un script de Bash
Un script de Bash es un archivo de texto plano que contiene una secuencia de comandos.

1. Shebang (#!/bin/bash): La primera linea del archivo. Indica al sistema operativo que interprete debe ejecutar el archivo.
2. Comentarios (#): Explicaciones del codigo para hacer el script reproducible.
3. Control de fallos (set -e): Detiene el script si cualquier comando falla.

Ejemplo de script:
```bash
#!/bin/bash
# SCRIPT: qc_pipeline.sh
# DESCRIPCION: Conteo y reporte basico de calidad para archivos FASTQ

set -e

echo "Inicio del control de calidad"
```

### 4.2. Variables posicionales
Los scripts pueden recibir parametros desde la linea de comandos al ejecutarse.

- $1: Primer argumento pasado al script.
- $2: Segundo argumento.
- $@: Todos los argumentos pasados.
- $#: Numero total de argumentos pasados.

Ejemplo:
```bash
#!/bin/bash
ARCHIVO_ENTRADA=$1
DIRECTORIO_SALIDA=$2

echo "Procesando archivo: $ARCHIVO_ENTRADA"
echo "Guardando resultados en: $DIRECTORIO_SALIDA"
```

### 4.3. Dar Permisos de Ejecucion (chmod +x)
Por defecto, los archivos creados no tienen permiso de ejecucion por razones de seguridad.

Comandos:
```bash
chmod +x qc_pipeline.sh
```
--------------------------------------------------------------------------------
### Ejercicio 4

Crea un script ejecutable llamado resumen_fasta.sh que:
1. Reciba como argumento ($1) la ruta a un archivo en formato FASTA (.fasta o .fa).
2. Cuente el numero de secuencias hay en el archivo (contando cuantas lineas empiezan por '>').
3. Imprima el resultado en la terminal.
--------------------------------------------------------------------------------

## Buenas practicas en bioinformatica

1. Nunca modifiques tus datos crudos (Raw Data): Mantén los archivos FASTQ/FASTA crudos en modo de solo lectura o en carpetas protegidas.
2. Documenta tus scripts: Incluye siempre encabezados con el autor, la fecha y el proposito del script.
3. Usa nombres descriptivos: Prefiere variables como MUESTRA_R1 sobre nombres genericos como X o A.
4. Control de Versiones: Almacena y comparte tus scripts de Bash en repositorios de GitHub o GitLab para garantizar la reproducibilidad de tus analisis doctorales.
