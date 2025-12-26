# DynamicsCustomAgent
Una forma interesante de sacarle partido al SidePane de Dynamics, incrustando un agente custom creado con Copilot Studio en Dynamics 365. Pasos para conseguirlo:

**1. Variable de entorno.**

Usa una variable de entorno para almacenar la información de configuración del agente en formato JSON:

- url: dirección del agente de copilot studio.
- name: nombre del agente.
- icon: icono que aparecerá en el panel.
- canClose: si permitimos poder cerrar el panel.
- expanded: si queremos que inicie expandido.
- width: tamaño de la ventana del panel expandido.

  **Nota:** se pueden añadir varios agentes, ya que es un array de JSON.
  
<pre>
[
  {
    "url":"https://copilotstudio.microsoft.com/environments/xxxxxxxxxxx", 
    "name": "XXXXXXXX", 
    "icon": "WebResources/xxxxxxx", 
    "canClose": false, 
    "expanded": false, 
    "width": 350
  }
]
</pre>

**2. HTML**

Crea un Webresource de tipo HTML que pinte un iframe, e inicialice el atributo src del iframe con la información pasada como parametro en la propiedad data de la URL.

```
<html style="height: 97%;">
    <header>
        <script>

            window.onload = function() {

                const queryString = window.location.search;
                const urlParams = new URLSearchParams(queryString);
                const data = urlParams.get('data');

                if (data) {
                    document.getElementById("miIframe").src = data;
                }
                else {
                    console.error("No data provided in the URL.");
                }
            };    
        </script>
    </header>
    <body  style="height: 100%;">
        <iframe id="miIframe" frameborder="0" style="width: 100%; height: 100%;"></iframe>
    </body>
</html>
```

**3. JavaScript**

Crea un webresource de tipo javaScript que lea la variable de entorno definida y genere tantos paneles como JSON encuentre. Pasando en la navegación del panel la configuración:

- ***pageType: "webresource"***
- ***webresourceName: xxxxxxxx*** (nombre del HTML creado en el paso anterior)
- ***data: element.url*** (es la URL del agente configurada en la variable de entorno)

<pre>
async function showGlobalMessage()
{
    var urls = await getEnvironmentVariableValue("xxxxxxxxxx"); //sustituir xxxxxxx por el nombre interno de la variable de entorno creada en el paso 1
    var cont = 0;
    urls.forEach(element => {
        var Pane = Xrm.App.sidePanes.getPane("Pane"+cont);

        if (Pane == null && element.url != "") {
            Xrm.App.sidePanes.createPane({
                title: element.name ?? "",
                imageSrc: element.icon,
                canClose: element.canClose ?? false,
                width: element.width ?? 350,
                paneId: "Pane"+cont,
                isSelected: element.expanded ?? false
            })
            .then((pane) => {
                pane.navigate({
                    pageType: "webresource",
                    webresourceName: "xxxxxxxxx.html", //sustituir xxxxxxx por el nombre del HTML creado en el paso 2
                    data: element.url
                });
            });
        } else if (element.url == ""){
            console.log("La URL del agente está vacía.");
        }
        cont++;
    });

    return false; 
}

async function getEnvironmentVariableValue(schemaName) {
    const entityVariableResults = await parent.Xrm.WebApi.retrieveMultipleRecords("environmentvariabledefinition", "?$filter=schemaname eq '"+schemaName+"'&$select=environmentvariabledefinitionid&$expand=environmentvariabledefinition_environmentvariablevalue($select=value)");

    if (!entityVariableResults || !entityVariableResults.entities || entityVariableResults.entities.length < 1) return null;
    const environmentVariableValue = entityVariableResults.entities[0];
    if (!environmentVariableValue.environmentvariabledefinition_environmentvariablevalue || environmentVariableValue.environmentvariabledefinition_environmentvariablevalue.length < 1) return null;

    var urls = JSON.parse(environmentVariableValue.environmentvariabledefinition_environmentvariablevalue[0].value);

    return urls;
}
</pre>

**4. Trigger**

Para que todo esto funcione necesitamos un trigger que llame a nuestra función cuando se cargue Dynamics, para ello:

1. Crearemos una solución que incluya el componente "Cintas de opciones de la aplicación" y lo abriremos con el plugin RibbonWorkbench del XRMToolbox.
2. Desde RibbonWorkbench:
   - Crearemos un botón, en el área Home buscaremos la sección ***Mscrm.GlobalTab*** y lo insertaremos.
   - Crearemos una Enable Rule de tipo CustomRule donde llamaremos a nuestra funcion js definida en el paso 3. Importante dejar el Default a False para que el botón este oculto.
     <img width="1072" height="176" alt="image" src="https://github.com/user-attachments/assets/279bdd1a-cb29-4423-b241-959247b7fbcf" />
   - Crear Command para enlazarlo al botón.
     <img width="1791" height="431" alt="image" src="https://github.com/user-attachments/assets/e4e9af93-7a05-4fe8-be9a-16288035c221" />
  
Con esto tras publicar todo en Dynamics y recargar tendremos los agentes que definidos en el panel lateral o side panel y podremos hacer uso de ellos.
