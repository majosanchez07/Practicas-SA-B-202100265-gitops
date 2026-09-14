# Repositorio GitOps — Práctica 8

**Manifiestos declarativos del sistema de microservicios de biblioteca.**

María José Tebalán Sánchez — 202100265
Software Avanzado B — Universidad de San Carlos de Guatemala

---

## Qué es este repositorio

Este repositorio es **la única fuente de verdad** sobre lo que se ejecuta en el
clúster. No contiene código de aplicación: solo la declaración de qué versión
de cada componente debe estar desplegada y con qué configuración.

El repositorio de código vive aparte:
[Practicas-SA-B-202100265](https://github.com/majosanchez07/Practicas-SA-B-202100265)

## Por qué está separado del código

Tres razones, en orden de importancia:

**1. El pipeline no tiene credenciales del clúster.** En la Práctica 7, el
pipeline ejecutaba `helm upgrade` contra el clúster: era administrador, de modo
que comprometer el repositorio implicaba comprometer la infraestructura. Aquí
el pipeline solo puede abrir un Pull Request en este repositorio. Quien aplica
los cambios es ArgoCD, que vive *dentro* del clúster y tira (pull) de aquí.

**2. Ciclos de vida distintos.** El código cambia cuando se programa; los
manifiestos cambian cuando se despliega. Mezclarlos obliga a filtros por ruta
frágiles, y cada commit de código dispararía una reconciliación.

**3. Sin bucles.** El pipeline escribe el nuevo tag aquí. Si fuera el mismo
repositorio, ese commit dispararía el pipeline otra vez, que escribiría otro
commit.

## Estructura

```
.
├── apps/                    Applications de ArgoCD (patrón app-of-apps)
├── environments/
│   ├── dev/values.yaml      Configuración del ambiente de desarrollo
│   └── prod/values.yaml     Configuración de producción — el pipeline
│                            modifica aquí UNA línea: imageTag
├── platform/
│   ├── kyverno/             Políticas de admisión
│   └── argo-rollouts/       Configuración del controlador de rollouts
└── docs/                    Evidencias de sincronización
```

## El flujo completo

```
  Repositorio de código                    Este repositorio
  ─────────────────────                    ────────────────
  git tag v1.0.1
        │
        ▼
  pipeline: build, test, helm lint
        │
        ▼
  Trivy  ──── CVE crítica ──► PR bloqueado
        │
        ▼
  SBOM + firma Cosign
        │
        └──── abre Pull Request ──────────►  environments/prod/values.yaml
                                                  imageTag: "1.0.1"
                                                         │
                                                    (merge)
                                                         │
                                                         ▼
                                              ArgoCD detecta el cambio
                                                         │
                                                         ▼
                                              Argo Rollouts: canary
                                              20% → 50% → 80% → 100%
                                                         │
                                            ┌────────────┴────────────┐
                                     análisis OK              análisis falla
                                            │                        │
                                            ▼                        ▼
                                      promoción              reversión automática
```

## Qué puede y qué no puede hacer el pipeline

| Acción | ¿Puede? |
|---|---|
| Cambiar `imageTag` en un Pull Request | Sí |
| Fusionar ese Pull Request | No |
| Aplicar manifiestos al clúster | No |
| Ejecutar `kubectl` o `helm upgrade` | No |
| Leer el kubeconfig | No lo tiene |

El token que el pipeline usa (`GITOPS_TOKEN`) tiene permiso de escritura
**solo sobre este repositorio**. Quien lo robara podría, como máximo, proponer
un cambio de versión — que seguiría pasando por el Pull Request, por las
políticas de admisión de Kyverno y por el análisis del canary.

## Verificar el estado

```bash
# ¿Está sincronizado con este repositorio?
kubectl get applications -n argocd

# Estado detallado
argocd app get sa-platform

# Historial de sincronizaciones
argocd app history sa-platform
```

## Revertir una versión

No hace falta tocar el clúster. Se revierte el commit:

```bash
git revert <commit-de-la-promocion>
git push
```

ArgoCD detecta el cambio y devuelve el clúster al estado anterior. Ése es el
sentido de que este repositorio sea la fuente de verdad: el `git log` de aquí
es el historial de lo que estuvo desplegado.
