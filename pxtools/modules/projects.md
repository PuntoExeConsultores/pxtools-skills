# Módulo @Projects — Gestión de Proyectos

> Comportamiento del módulo `@PXTools/@Projects`. Índice de módulos: [20-modulos-pxtools.md](../20-modulos-pxtools.md).

**Ubicación en la KB**
- Módulo: `Knowledge Base/@PXTools/@Projects` (`APIs/` core + `Personalized/`).
- Puente con seguridad: `Knowledge Base/@PXTools/@SecurityProjects/` (solo grupos de subtipos, ver §3).
- Cualificador: `PXTools.Projects`.
- **Depende de:** `@APIs` (base), `@Security` (usuarios/roles, vía los subtipos de `@SecurityProjects`), `@Menus`.

## 1. Qué provee

Gestión de **proyectos categorizados por tipo**, con **miembros** (usuarios) y **roles por miembro dentro de cada proyecto**. Se integra con @Security (usuarios/roles) para saber quién participa de cada proyecto y con qué rol.

## 2. Transacciones del módulo

| Transacción | PK | Rol |
|---|---|---|
| **Projects** | `ProjectId` | `ProjectProjectTypeId` (FK a ProjectTypes, nullable), `ProjectName`, `ProjectInitialDate` (default `Today()`), `ProjectDescription`, `ProjectEnabled`. |
| **ProjectTypes** | `ProjectTypeId` | `ProjectTypeName`, `ProjectTypeDescription`. |
| **ProjectsMembers** | `ProjectId, ProjectMemberSecurityUserId` | Miembros (usuarios) del proyecto. Regla `AfterComplete`: propaga alta/baja a `ProjectsMembersRoles` (`UpdProjectsMembersRoles`). |
| **ProjectsMembersRoles** | `ProjectId, ProjectMemberRolSecurityUserId, ProjectMemberRolSecurityRoleId` | Roles de cada miembro dentro del proyecto. |

```
ProjectTypes ──1:N──▶ Projects ──1:N──▶ ProjectsMembers ──▶ SecurityUsers
                         └────1:N──▶ ProjectsMembersRoles ──▶ SecurityUsers / SecurityRoles
```

## 3. Integración con @Security vía @SecurityProjects

El enlace con Security **no** es una FK directa: lo materializan **grupos de subtipos** en `@PXTools/@SecurityProjects/` (que es la **capa base** que integra este módulo con Security):
- `ProjectMemberSecurityUserId : SecurityUserId`
- `ProjectMemberRolSecurityUserId : SecurityUserId`
- `ProjectMemberRolSecurityRoleId : SecurityRoleId`

Así `ProjectsMembers`/`ProjectsMembersRoles` heredan `SecurityUsers`/`SecurityRoles` como tablas foráneas. Al asignar un miembro, el combo de roles se restringe a **`SecurityRoleType = SecurityRoleType.Project`**. Cada transacción se registra en el `SecurityContext` bajo el módulo `PXTools.Projects`.

## 4. Dominios del módulo

- **Propio:** `ProjectId` (`IdFirstLevel`, **root-legacy**) — único dominio `Project*`. No hay dominios enum propios.
- **Usa de otros módulos:** `SecurityRoleType` (valor `Project` para filtrar roles asignables), `SecurityUserId`, `SecurityRoleId` (@Security, vía subtipos); tipos base `ObjectName`, `ObjectDescription`, `Boolean` (@APIs/GeneXus).

## 5. APIs vs Personalized

- **`APIs/`** (core): las transacciones, `UpdProjectMembers`, `UpdProjectsMembersRoles`, `RetProjects`, `ChkProjectMembersExistance`.
- **`Personalized/`**: `RetMenuProjects` (DataProvider) — entradas de menú del módulo ("Project Types" / "Projects").

## 6. Instancias de patterns

| Instancia | Qué es |
|---|---|
| **PXWorkWithProjects** | WW completo (Selection + View). La View embebe la sección **"Project Members"** (`CmProjectsMembers`). |
| **PXWorkWithProjectTypes** | WW de tipos. |
| **PXWorkWithProjectsMembers** | **Component** embebido en la View de Projects (parm `ProjectId`); combos UserId (`RetUsers`) y RoleId (roles `Project`); Insert/Delete llaman `UpdProjectMembers`. |

## 7. Procedimientos / APIs clave

| Proc | `Parm()` | Propósito |
|---|---|---|
| `UpdProjectMembers` | `in: &ProjectId, &ProjectMemberSecurityUserId, &SecurityRoleId, &Mode` | Alta/baja de miembro (crea `ProjectsMembers` + delega el rol). Entry point del pattern de miembros. |
| `UpdProjectsMembersRoles` | `in: &Mode, &ProjectId, &…RolSecurityUserId, &SecurityRoleId` | Gestiona la fila en `ProjectsMembersRoles`. |
| `RetProjects` (DataProvider) | `in: &UserData` → `SDTObjects` | Proyectos donde el usuario es miembro. |
| `ChkProjectMembersExistance` | `out: &Exist` | ¿Existe al menos un miembro? |

## Referencias
- [20-modulos-pxtools.md](../20-modulos-pxtools.md) — índice de módulos.
- [security.md](security.md) — usuarios/roles (`SecurityUsers`/`SecurityRoles`); el rol asignable es `SecurityRoleType.Project`.
- Módulo **@SecurityProjects** — grupos de subtipos que integran Projects con Security.
- Módulo **@Menus** — la entrada de menú se declara en `RetMenuProjects`.
