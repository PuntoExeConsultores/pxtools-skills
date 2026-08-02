# Changelog

Registro de cambios de las skills de PXTools (documentación para IAs).

El formato sigue [Keep a Changelog](https://keepachangelog.com/es-ES/1.1.0/).
Este repositorio no versiona software sino documentación, por lo que las entradas se agrupan
por fecha de publicación en `master`.

## [Sin publicar]

Nada pendiente.

## 2026-08-01

### Agregado
- **`pxtools/modulos/menus.md`** — sección *5.1.1 `Parent`: SOLO en el nodo raíz de cada
  `RetMenus<X>`*. `Parent` engancha la porción de árbol que aporta el módulo debajo de un nodo
  existente (típicamente `'Basic'`) y va únicamente en el/los ítems del primer nivel; la jerarquía
  interna la da el anidamiento en `Childs`. Incluye el fragmento de `AddMenusRecursive` que lo
  demuestra (`Parent` se lee solo cuando no viene un padre por la recursión) y la advertencia de
  que `RetParentMenu` **crea** el menú si el nombre no existe, de modo que un `Parent` mal escrito
  en el nodo raíz no falla: genera silenciosamente un nodo huérfano.

## 2026-07-18

### Agregado
- `README.md`, `LICENSE` (CC BY 4.0) y `.gitignore`.

### Cambiado
- Documentación de módulos, dominios *root-legacy*, dependencias entre módulos y criterios de
  atribución de dominios a su módulo.
- Ejemplos genericizados: se reemplazaron nombres de objetos de clientes por nombres neutros.

## 2026-05-28

### Agregado
- Publicación inicial de las skills de PXTools: visión general del framework, patterns de UI
  (PXWorkWith, PXParameterRequest, PXComposer), de flujo (PXFlowController), de API/WebServices
  (PXWSLayer, PXWSQuery, PXWSData, PXWSTransaction), de datos/configuración (PXOAV,
  PXEntityParameters, PXReportTemplate), documentación por módulo `@PXTools/*` y guías
  transversales de reconocimiento y migración.

---

## Cómo escribir una entrada

- Agrupar por fecha de publicación en `master`, con las secciones de Keep a Changelog que
  correspondan: **Agregado**, **Cambiado**, **Obsoleto**, **Eliminado**, **Corregido**.
- Nombrar el archivo tocado y **qué** se documentó, no solo que "se documentó algo".
- Cuando la entrada corrige documentación previa que era incorrecta, decirlo explícitamente: quien
  la leyó antes necesita saber que cambió.
- Si el hallazgo se verificó contra una KB real o contra lo que emite el IDE, mencionarlo — separa
  lo verificado de lo deducido.
