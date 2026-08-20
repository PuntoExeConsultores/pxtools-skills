# @Projects Module — Project Management

> Behaviour of the `@PXTools/@Projects` module. Module index: [20-pxtools-modules.md](../20-pxtools-modules.md).

**Location in the KB**
- Module: `Knowledge Base/@PXTools/@Projects` (`APIs/` core + `Personalized/`).
- Bridge to security: `Knowledge Base/@PXTools/@SecurityProjects/` (subtype groups only, see §3).
- Qualifier: `PXTools.Projects`.
- **Depends on:** `@APIs` (base), `@Security` (users/roles, through the `@SecurityProjects` subtypes), `@Menus`.

## 1. What it provides

Management of **projects categorised by type**, with **members** (users) and **per-member roles inside each project**. It integrates with @Security (users/roles) to know who takes part in each project and in what role.

## 2. Module transactions

| Transaction | PK | Role |
|---|---|---|
| **Projects** | `ProjectId` | `ProjectProjectTypeId` (FK to ProjectTypes, nullable), `ProjectName`, `ProjectInitialDate` (defaults to `Today()`), `ProjectDescription`, `ProjectEnabled`. |
| **ProjectTypes** | `ProjectTypeId` | `ProjectTypeName`, `ProjectTypeDescription`. |
| **ProjectsMembers** | `ProjectId, ProjectMemberSecurityUserId` | Project members (users). `AfterComplete` rule: propagates additions/removals to `ProjectsMembersRoles` (`UpdProjectsMembersRoles`). |
| **ProjectsMembersRoles** | `ProjectId, ProjectMemberRolSecurityUserId, ProjectMemberRolSecurityRoleId` | Each member's roles inside the project. |

```
ProjectTypes ──1:N──▶ Projects ──1:N──▶ ProjectsMembers ──▶ SecurityUsers
                         └────1:N──▶ ProjectsMembersRoles ──▶ SecurityUsers / SecurityRoles
```

## 3. Integration with @Security through @SecurityProjects

The link to Security is **not** a direct FK: it is materialised by **subtype groups** in `@PXTools/@SecurityProjects/` (the **base layer** integrating this module with Security):
- `ProjectMemberSecurityUserId : SecurityUserId`
- `ProjectMemberRolSecurityUserId : SecurityUserId`
- `ProjectMemberRolSecurityRoleId : SecurityRoleId`

That is how `ProjectsMembers`/`ProjectsMembersRoles` inherit `SecurityUsers`/`SecurityRoles` as foreign tables. When assigning a member, the roles combo is restricted to **`SecurityRoleType = SecurityRoleType.Project`**. Each transaction registers itself in the `SecurityContext` under the `PXTools.Projects` module.

## 4. Module domains

- **Its own:** `ProjectId` (`IdFirstLevel`, **root-legacy**) — the only `Project*` domain. There are no enumerated domains of its own.
- **Used from other modules:** `SecurityRoleType` (the `Project` value, to filter assignable roles), `SecurityUserId`, `SecurityRoleId` (@Security, through subtypes); base types `ObjectName`, `ObjectDescription`, `Boolean` (@APIs/GeneXus).

## 5. APIs vs Personalized

- **`APIs/`** (core): the transactions, `UpdProjectMembers`, `UpdProjectsMembersRoles`, `RetProjects`, `ChkProjectMembersExistance`.
- **`Personalized/`**: `RetMenuProjects` (DataProvider) — the module's menu entries ("Project Types" / "Projects").

## 6. Pattern instances

| Instance | What it is |
|---|---|
| **PXWorkWithProjects** | Full WW (Selection + View). The View embeds the **"Project Members"** section (`CmProjectsMembers`). |
| **PXWorkWithProjectTypes** | WW for the types. |
| **PXWorkWithProjectsMembers** | **Component** embedded in the Projects View (parm `ProjectId`); UserId (`RetUsers`) and RoleId (`Project` roles) combos; Insert/Delete call `UpdProjectMembers`. |

## 7. Key procedures / APIs

| Proc | `Parm()` | Purpose |
|---|---|---|
| `UpdProjectMembers` | `in: &ProjectId, &ProjectMemberSecurityUserId, &SecurityRoleId, &Mode` | Add/remove a member (creates the `ProjectsMembers` row and delegates the role). Entry point of the members pattern. |
| `UpdProjectsMembersRoles` | `in: &Mode, &ProjectId, &…RolSecurityUserId, &SecurityRoleId` | Manages the row in `ProjectsMembersRoles`. |
| `RetProjects` (DataProvider) | `in: &UserData` → `SDTObjects` | The projects the user is a member of. |
| `ChkProjectMembersExistance` | `out: &Exist` | Is there at least one member? |

## References
- [20-pxtools-modules.md](../20-pxtools-modules.md) — module index.
- [security.md](security.md) — users/roles (`SecurityUsers`/`SecurityRoles`); the assignable role is `SecurityRoleType.Project`.
- The **@SecurityProjects** module — subtype groups integrating Projects with Security.
- The **@Menus** module — the menu entry is declared in `RetMenuProjects`.
