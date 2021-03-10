# Objetivo
Replicar los datos desde una instacia local hospitalaria a un concetrador o viceversa.

## Vista General: Es decir. funcionalidad deseada:
 * Trabajar sobre una lista de tablas a replicar configuradas fijamente desde el momento de la creacion del modulo/ejecutable empaquetador
 * Las tablas dichas deberan tener una columna tipo fecha_hora con el dato de cuando fue insertado o modificado por ultima vez
 * Los borrados seran lógicos (el registro debera permanecer en la tabla con campo de status que lo marque como ***INACTIVO***)
 * Ademas de las tablas a replicar habra una tabla *PAQUETES_ACTUALIZACION* que contendra la lista de
   paquetes que ya se han generado con minimos los siguientes campos
    * fecha: fecha de generacion del paquete
    * nombre: nombre del paquete
    * unidadMedicaID
    * estado: enumeracion entre: *GENERADO, APLICADO, o INACTIVADO*
 * Esta tabla se incluira integra con cada paquete que se genere

## Contenidos y estructura del paquete
 * se generara en un archivo en formato ZIP con parametro de compresion maxima (prefiriendo espacio sobre velocidad) y encriptado con una contraseña ya sea definida previamente o durante la generacion del paquete mismo
 * este archivo incluira:
   * Cambios.json
     * este archivo llevara por cada tabla a replicar los registros completos cuya fecha de modificacion sea mayor o igual a la ultima fecha de generacion de paquete. asi como el dump completo de la tabla *PAQUETES_ACTUALIZACION*.

      Formato:

        ```json
        {
          "fecha":"&lt;fecha de generacion del paquete&gt;",
          "Cambios": [{"Tabla": "<nombre_de_tabla>","Rows": [{"campo1": "valor1","campo2": "valor2"...}...]}, {
            "Tabla": "<nombre_de_tabla #2>",
            "Rows": [{
              "campo1": "valor1",
              "campo2": "valor2"...
            }...]
          }],
          "Paquetes": "Rows": [{
            "campo1": "valor1",
            "campo2": "valor2"...
          }...]
        }
        ```

   * Firmas.txt:
     * Para Verificar tanto la autenticidad como la integridad del paquete:
       * se calculara Primero el HASH criptografico ***SHA-512 (SHA-2)*** del archivo Cambios.json llamemosle **H1**
       * mediante una Clave Privada Asymetrica ***RSA*** de un minimo de 2048 bits (recomendado 4096) se calculara la firma del *HASH* resultante de la operacion anterior. llamemosle **F1**
     * el archivo incluira:
       * **GENERADO**:&lt;fecha de generacion en formato *ISO8601* sin timezone (local) YYYY-MM-DDTHH:mm:ss &gt;**POR**:&lt;Nombre de Usuario que genero el paquete - Unidad Medica&gt;
       * BASE64(**H1**):BASE64(**F1**)
       * linea en blanco (opcional)

     Ejemplo *(notese que esto no son datos validos, solo para caracter ilustrativo)*:

     ```
     GENERADO:2015-10-28T10:47:00 POR: RSALITRERO - HOSPITAL CENTRAL
     z4PhNX7vuL3xVChQ1m2AB9Yg5AULVxXcg/SpIdNs6c5H0NE8XYXysP+DGNKHfuwvY7kxvUdBeoGlODJ6+SfaPg==:F1S8YOX8iPviy1p02566+uEIfuFRMGNqJ1KXDISBWSGZ+QeP7+pLPecZmOl8XBxGZjjLPB375XrTtv7dGNbn4o1/roh3Y2O74GHgI71ldDiA0xFYK8OSLT6shHOl/dFjtFHKa+oQooUcxJ4pVEVa/G1iYEvEnHgeHk3OnSnsvWE+cPd5eXX9SCAy9RaA+p87Q+duUEJaeAyrUkaYjDNRyGh/6WyVAWPduRG3Me9XgS8d6JtNA7Wx/WckJnu0aR5ripHAdruwNl5x6FwKMHg6RYmUG5vtJoR729GPaGqOreohxtLhs2ENvLNagwz9BOryijP4NgVXSBfgmBmyU0eNYw==

     ```

## Razones:
 * Para el formato ZIP:
   * se propone debido a su amplia disponibilidad de librerias y utilidades      que permiten su manejo y debido a que en pruebas de compresion de texto ha demostrado una capacidad suficiente de reduccion de tamaño lograndose en algunos caso hasta un 70%-80% de compresion
   * Encriptacion: El formato zip permite encriptar los contenidos con el estandar AES-256 lo que aseguraria (siempre y cuando la contraseña sea lo bastante segura) que aun en el caso del robo de un paquete por una tercera parte este no podria ser usado para obtener informacion privada.
 * Para el HASH y la firma:
   * El estandar SHA-512 es actualmente considerado altamente seguro para verificar la integeridad de un archivo aun ante ataques de colision supeditando y reemplazando el ya obsoleto e inseguro SHA-1.

     Este genera un digest o hash de 512 bytes para cualquier tamaño de mensaje (archivo) a tratar desde pocos bytes hasta más allá de los PetaBytes.

     Para verificar la integridad del archivo basta con calcular otra vez el SHA-512 de Cambios.json y debe coincidir con *H1* en el archivo Firmas.txt. si los resultados no son identicos el archivo se considera corrupto y **NO DEBERA SER USADO**
   * Del mismo modo la firma RSA se calcula sobre el HASH y para verificarla se utiliza la Clave Publica correspondiente a La Llave Privada con que se generó el paquete
     * la llave Privada **DEBE SOLO ESTAR ALMACENADA EN DONDE SE GENERO EL PAQUETE Y NUNCA SE COMPARTE**.
     * la llave publica **DEBE ESTAR EN DONDE SE DESEE VERIFICAR EL ARCHIVO, PUEDE COMPARTIRSE LIBREMENTE SIN NINGUN RIESGO**.

    La verificacion consiste en correr el proceso inverso a la firma. se decrypta la firma generada **F1** con la **LLAVE PUBLICA** Y el resultado debe ser el hash **H1** esto comprueba que efectivamente **H1** fue firmado por la **LLAVE PRIVADA** autorizada y no por un tercero que quisiera insertar datos anomalos o ilegales en el destino

    Nuevamente si el resultado no coincide **NO DEBERA SER UTILIZADO PUESTO QUE HA SIDO ALTERADO**

## Sobre el proceso de actualizacion
  * Una vez que el archivo ha sido validado se cargara la tabla de paquetes y se actualizara respecto a la existente en el BD de destino.
  * Se verificara que el paquete que se desea instalar sea el último en la lista de cambios (puesto que son sequenciales) verificando la columna de estado si falta algun paquete intermedio se detendra la operacion y se avisara al usuario con la lista de paquetes faltantes por aplicar
  * Superado este paso por cada registro de cada tabla encontrado en el paquete se aplicaran en orden de integridad referencial los cambios.
    * Se generara una lista (basado en los ids) de los registros faltantes y de los registros ya existentes.
      * Los faltantes se insertaran
      * Los Existentes se actualizaran para dejar todos los campos identicos al paquete
    * si llega a ocurrir un error se continuara con el siguiente registro guardando los resultados malos en un archivo Errores.txt el cual incluira tabla,registro y mensaje de error.
  * se marcara en la tabla de paquetes del destino al paquete como *APLICADO*
  * finalmente de generará un paquete minimo para aplicar en el origen que solo llevara los cambios de la tabla de paquetes ese lleara el mismo formato que el paquete usado (solamente que viene generado del concentrador y firmado por la *LLAVE PRIVADA* del concentrador el proceso de aplicacion es el mismo) esto es para tener una idea clara de que paquetes han sido aplicados en el destino

## Si llegase a faltar un paquete secuencial:
  * En el origen se podra marcar cual es el paquete que falta y solicitar la creacion de un nuevo paquete.
  * todos los paquetes a partir del que fue extraviado seran marcados como *INACTIVADO*
  * el nuevo paquete se calculara como si trajera los cambios desde el ultimo paquete *GENERADO* O *APLICADO* en el origen y llevara todos los cambios de la misma manera que se generan siempre.
  * los registros INACTIVADOS son ignorados a la hora de verificar la aplicacion secuencial de paquetes.
