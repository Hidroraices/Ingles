# Seguridad del piloto — Control de Paradas

## Objetivo
Validar acceso gerencial desde fuera de planta sin publicar información operacional como recurso abierto de Internet.

## Control seleccionado para el piloto
Acceso privado autenticado mediante una red privada cifrada. El dashboard continúa ejecutándose en la central de planta y se presenta mediante HTTPS únicamente a usuarios/dispositivos autorizados.

## Frontera de seguridad
- No usar publicación pública.
- No usar port-forwarding del router.
- No exponer directamente las PCs TL1/TL2/TL3.
- No subir Excel, SQLite ni datos productivos reales a GitHub.
- No incluir credenciales o rutas internas en el repositorio.
- Los archivos fuente permanecen en planta.

## Datos visibles
La publicación gerencial debe limitarse a datos necesarios para análisis. Por defecto se excluyen nombres de trabajadores, rutas internas, nombres físicos de archivos, credenciales e información de infraestructura.

## Prueba mínima de aceptación
1. El gerente autorizado puede abrir el dashboard desde fuera de planta.
2. Un dispositivo/usuario no autorizado no puede abrirlo.
3. El dashboard no expone Excel ni SQLite.
4. No existe acceso entrante directo desde Internet hacia las PCs de los trenes.
5. El tráfico externo usa HTTPS/canal cifrado.

## Escalamiento
El piloto debe elevarse a TI/Seguridad de Información antes de convertirse en una solución corporativa permanente. La capa privada del piloto puede ser reemplazada posteriormente por VPN/SSO/Zero Trust corporativo sin cambiar la lógica de captura.
