# Instalación y Validación del Esquema HR con XML en Oracle

![Oracle](https://img.shields.io/badge/Oracle-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![SQL](https://img.shields.io/badge/SQL-4479A1?style=for-the-badge&logo=postgresql&logoColor=white)
![PL/SQL](https://img.shields.io/badge/PL/SQL-F80000?style=for-the-badge&logo=oracle&logoColor=white)
![GitHub](https://img.shields.io/badge/GitHub-100000?style=for-the-badge&logo=github&logoColor=white)

> **Proceso completo de instalación del esquema HR en Oracle 21c XE, generación de XML y validación mediante triggers y XSD.**

## 📋 Tabla de Contenidos

- [Sobre este proyecto](#sobre-este-proyecto)
- [Requisitos previos](#requisitos-previos)
- [Instalación del esquema HR](#instalación-del-esquema-hr)
- [Generación de XML desde EMPLOYEES](#generación-de-xml-desde-employees)
- [Registro del esquema XML (XSD)](#registro-del-esquema-xml-xsd)
- [Trigger de validación](#trigger-de-validación)
- [Pruebas de validación](#pruebas-de-validación)
  - [Inserción válida](#inserción-válida)
  - [Inserción inválida](#inserción-inválida)
- [Tecnologías utilizadas](#tecnologías-utilizadas)
- [Archivos incluidos](#archivos-incluidos)
- [Autor](#autor)

---

## Sobre este proyecto

Este repositorio documenta el proceso completo para:

- **Instalar** el esquema de ejemplo `HR` (Human Resources) en Oracle Database 21c XE.
- **Generar** documentos XML a partir de los datos de la tabla `EMPLOYEES`.
- **Registrar** un esquema XML (XSD) que define la estructura esperada.
- **Implementar** un trigger en PL/SQL que valida automáticamente las inserciones en la tabla `XML_TAB`.
- **Probar** la validación con inserciones válidas e inválidas.

---

## Requisitos previos

- Oracle Database 21c XE instalado.
- Usuario con privilegios de administrador (`SYSTEM` o `SYS`).
- Acceso a SQL*Plus o SQL Developer.
- Conexión al contenedor `XEPDB1`.

---

## Instalación del esquema HR

### Descargar el archivo `hr.zip`

Descarga el archivo desde el [sitio oficial de Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/21/comsc/installing-sample-schemas.html#GUID-1E645D09-F91F-4BA6-A286-57C5EC66321D) o desde su [repositorio oficial en GitHub](https://github.com/oracle-samples/db-sample-schemas/releases/tag/v23.3).

### Descomprimir y copiar

Descomprime el archivo y copia la carpeta `human_resources` en la siguiente ruta:

```bash
C:\app\TU_USUARIO\product\21c\dbhomeXE\demo\schema\

# Reemplaza `TU_USUARIO` con tu nombre de usuario de Windows
```

## Ejecutar el script de instalación

**Abre SQL*Plus** como administrador:

```sql
sqlplus / as sysdba
ALTER SESSION SET CONTAINER = XEPDB1;
@?/demo/schema/human_resources/hr_install.sql;
```

**El instalador te pedira que respondas las siguiente preguntas**

<img width="771" height="541" alt="Imagen1" src="https://github.com/user-attachments/assets/9a407556-0f0c-4f91-bc6d-f1a45d8c59d7" />

| Pregunta |	Respuesta |
|----------|------------|
| Password para HR |	hr |
| Tablespace |	USERS (presiona Enter) |
| Sobrescribir |	YES |

## Generación de XML desde EMPLOYEES

**Terminada la instalacio**, el siguiente paso es crear la Tabla para XML para almacenar los documentos XML

```sql
-- Crear la tabla XML_TAB
CREATE TABLE xml_tab (
    id        NUMBER,
    xml_data  XMLTYPE
);
```

<img width="1085" height="651" alt="Imagen2" src="https://github.com/user-attachments/assets/8ba51b36-5bd2-4ca4-b884-eba0ad4bdc22" />

## Generar el XML

Este bloque PL/SQL genera un documento XML con los datos de los empleados y lo inserta en XML_TAB:

```sql
-- Generar el XML con los empleados (usando la tabla EMPLOYEES)
DECLARE
    l_xmltype XMLTYPE;
BEGIN
    SELECT XMLELEMENT("employees",
               XMLAGG(
                   XMLELEMENT("employee",
                       XMLFOREST(
                           e.employee_id AS "empno",
                           e.first_name || ' ' || e.last_name AS "ename",
                           e.job_id AS "job",
                           TO_CHAR(e.hire_date, 'DD-MON-YYYY') AS "hiredate"
                       )
                   )
               )
           )
    INTO l_xmltype
    FROM employees e;

    -- Insertar el XML generado en la tabla xml_tab 
    INSERT INTO xml_tab VALUES (1, l_xmltype);
    COMMIT;
END;
/
```

<img width="537" height="652" alt="imagen3" src="https://github.com/user-attachments/assets/8ef52642-f21d-40af-8e98-de72fe2d68d8" />

El archivo XML generado puede descargarse desde SQL Developer o visualizarse en tu IDE favorito (ej. Visual Studio Code):

<img width="651" height="873" alt="imagen4" src="https://github.com/user-attachments/assets/847bedc1-3be5-4382-999b-d37fc4023286" />

## Registro del esquema XML (XSD)

Registramos el esquema employees.xsd en la base de datos para definir la estructura esperada:

```sql
-- Registrar el esquema XML 'employees.xsd'
BEGIN
    DBMS_XMLSCHEMA.registerSchema(
        SCHEMAURL => 'employees.xsd',
        SCHEMADOC => '<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           xmlns:xdb="http://xmlns.oracle.com/xdb">
    <xs:element name="employees" xdb:SQLType="EMPLOYEES_XML_T">
        <xs:complexType xdb:SQLType="EMPLOYEES_XML_T">
            <xs:sequence>
                <xs:element name="employee" type="EmployeeType" 
                            minOccurs="0" maxOccurs="unbounded"
                            xdb:SQLCollType="EMPLOYEE_COLL_T"/>
            </xs:sequence>
        </xs:complexType>
    </xs:element>
    <xs:complexType name="EmployeeType" xdb:SQLType="EMPLOYEE_XML_T">
        <xs:sequence>
            <xs:element name="empno" type="xs:integer"/>
            <xs:element name="ename" type="xs:string"/>
            <xs:element name="job" type="xs:string"/>
            <xs:element name="hiredate" type="xs:string"/>
        </xs:sequence>
    </xs:complexType>
</xs:schema>',
        LOCAL => TRUE,
        GENTYPES => TRUE
    );
END;
/
```

<img width="761" height="752" alt="imagen5" src="https://github.com/user-attachments/assets/cfa1227f-3f29-4636-baff-2ab30eb9f3f6" />

## Trigger de validación

Este trigger se ejecuta antes de cada inserción o actualización en XML_TAB y valida que el XML cumpla con la estructura definida:  

```sql
-- Crear el trigger con validación manual (usando XMLTABLE)
CREATE OR REPLACE TRIGGER trg_validar_xml_tab
BEFORE INSERT OR UPDATE ON xml_tab
FOR EACH ROW
DECLARE
    v_count_employees NUMBER;
    v_count_valid NUMBER;
BEGIN
    -- Contar cuántos elementos 'employee' tiene el XML
    SELECT COUNT(*)
    INTO v_count_employees
    FROM XMLTABLE('/employees/employee' PASSING :NEW.xml_data);
    
    -- Si no hay elementos 'employee', el XML está vacío o mal formado
    IF v_count_employees = 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'El XML no contiene elementos "employee"');
    END IF;

    -- Validar que cada 'employee' tenga el campo 'ename'
    SELECT COUNT(*)
    INTO v_count_valid
    FROM XMLTABLE('/employees/employee' 
        PASSING :NEW.xml_data
        COLUMNS
            ename VARCHAR2(50) PATH 'ename'
    )
    WHERE ename IS NULL;

    -- Si algún empleado no tiene 'ename', lanzamos error
    IF v_count_valid > 0 THEN
        RAISE_APPLICATION_ERROR(-20001, 'Error: Existen empleados sin el campo <ename>');
    END IF;

EXCEPTION
    WHEN OTHERS THEN
        -- Capturar cualquier otro error
        RAISE_APPLICATION_ERROR(-20001, 'El XML no es válido: ' || SQLERRM);
END;
/
```

<img width="790" height="776" alt="imagen6" src="https://github.com/user-attachments/assets/4ca1f727-68ee-45b9-ac5b-32148ce046bc" />

## Pruebas de validación

### Inserción válida

```sql
-- Insertar un XML que SÍ cumple con la estructura
INSERT INTO xml_tab VALUES (
    2,
    XMLTYPE('<?xml version="1.0" encoding="UTF-8"?>
<employees>
    <employee>
        <empno>999</empno>
        <ename>Primo Valido</ename>
        <job>PRUEBA</job>
        <hiredate>01-JAN-2024</hiredate>
    </employee>
</employees>')
);

-- Verificar que se insertó
SELECT * FROM xml_tab WHERE id = 2;
```
<img width="763" height="397" alt="imagen8" src="https://github.com/user-attachments/assets/5c207f4f-4187-4b81-99cf-1a860a9318b2" />


### Inserción inválida (falta <ename>)

```sql
-- Insertar un XML que NO cumple con la estructura (falta 'ename')
INSERT INTO xml_tab VALUES (
    3,
    XMLTYPE('<?xml version="1.0" encoding="UTF-8"?>
<employees>
    <employee>
        <empno>888</empno>
        <!-- Falta el campo <ename> -->
        <job>PRUEBA_INVALIDA</job>
        <hiredate>01-JAN-2024</hiredate>
    </employee>
</employees>')
);

-- Esto DEBE fallar con el error: 
-- Error SQL: ORA-20001: El XML no es válido: ORA-20001: Error: Existen empleados sin el campo <ename>
```

<img width="892" height="478" alt="imagen7" src="https://github.com/user-attachments/assets/e6e5c0ff-d710-4539-b027-f060862b719e" />

## Tecnologías utilizadas

| Herramienta |	Descripción |
|-------------|-------------|
| Oracle Database 21c XE |	Sistema gestor de bases de datos |
| SQL*Plus |	Cliente de línea de comandos para Oracle |
| PL/SQL |	Lenguaje procedural de Oracle |
| XML |	Lenguaje de marcado para datos estructurados |
| XSD |	Definición de esquema XML |
| GitHub |	Control de versiones y documentación |

## Archivos incluidos
- EMPLOYEESXML
- pruebaDeInsersionEschemaInvalido
- pruebaDeInsersionEschemaValido
- schema.xsd

## Autor
Humberto Isaac Padilla Contreras
- GitHub: <a href="https://github.com/LovecraftianCode">@LovecraftianCode<a/>
- LinkedIn: <a href="https://www.linkedin.com/in/humberto-isaac-padilla-contreras-3527aa3b7" target="_blank">Humberto Isaac Padilla Contreras</a>
