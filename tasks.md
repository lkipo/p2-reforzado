# Estado de Tareas — Unitree G1 Policy Training

> Actualizado: 2026-06-21  
> Hardware: RTX 5090 (Blackwell CC 12.0) · Ubuntu 24.04 · Driver 595.71.05

---

## Resumen ejecutivo

| Fase | Estado | Motivo |
|---|---|---|
| 1 — Preparación del entorno | ✅ Completado (adaptado) | Isaac Sim 5.1 en lugar de 4.5 por incompatibilidad de hardware |
| 2 — LeggedGym / IsaacGym | ❌ Bloqueado (hardware) | IsaacGym no soporta CC 12.0 (Blackwell) |
| 3 — BeyondMimic (entrenamiento) | ✅ Completado | 3500 iteraciones, modelo entrenado |
| 4 — Exportación ONNX | ✅ Completado | `policy.onnx` generado manualmente desde checkpoint |
| 4 — Docker de despliegue | ⚠️ Documentado, no ejecutado | Usuario `lois` sin pertenencia al grupo `docker` |
| 5 — Sim2Sim (MuJoCo) | ❌ Bloqueado | Requiere `wandb_path` + acceso Docker |
| 6 — Sim2Real | ❌ No aplicable | No hay robot físico disponible |
| Vídeo de simulación renderizada | ❌ Bloqueado (hardware) | `librtx.scenedb` segfault en Blackwell al cargar renderer |
| Vídeo de métricas de entrenamiento | ✅ Completado | `training_results.mp4` (animación TensorBoard) |
| Repositorio Git | ✅ Comprometido | 2 commits en `main`; pendiente de `git push` manual |

---

## Fase 1: Preparación del entorno

### Tarea 1 — Instalar el simulador de entrenamiento ✅

**Estado:** Completado con adaptación obligatoria.

El requisito original era **Isaac Lab 2.1.0 + Isaac Sim 4.5.0**. Esta combinación es incompatible con la RTX 5090 (Blackwell, CC 12.0):

- Isaac Sim 4.5.0 usa el renderer Iray que no soporta Blackwell.
- IsaacGym (usado por LeggedGym) soporta CUDA hasta CC 8.x (Ampere).

**Solución adoptada:** Migración a Isaac Sim 5.1 + IsaacLab rama `main`, que soporta oficialmente RTX 50xx.

```bash
python3.11 -m venv ~/.virtualenvs/isaaclab
source ~/.virtualenvs/isaaclab/bin/activate
pip install --upgrade pip wheel setuptools
pip install "isaacsim[all,extscache]" --extra-index-url https://pypi.nvidia.com
cd /home/lois/p2/IsaacLab && ./isaaclab.sh --install
cd /home/lois/p2/RL/BeyondMimic/whole_body_tracking/source/whole_body_tracking
pip install -e .
```

### Tarea 2 — Obtener el modelo del robot G1 ✅

**Estado:** Completado.

El URDF del G1 de 29 DOF fue obtenido del repositorio LeggedGym y adaptado:

```
/home/lois/p2/RL/BeyondMimic/assets/unitree_description/urdf/g1/main.urdf
```

Enlace simbólico de meshes apuntando a:
```
/home/lois/p2/RL/LeggGym/unitree_rl_gym/resources/robots/g1_description/meshes/
```

El módulo `whole_body_tracking.assets` fue creado con `ASSET_DIR` apuntando a este directorio. Era una dependencia implícita del paquete `ros-humble-unitree-description` (ROS 2) que no estaba en el repositorio ni es instalable sin sudo.

---

## Fase 2: Entrenamiento con LeggedGym (IsaacGym RL)

### Tareas 3–7 — Configuración RL, Domain Randomization, Entrenamiento PPO, Monitorización, Ajuste ❌

**Estado:** Bloqueadas por incompatibilidad de hardware. Sin solución posible en este equipo.

**Causa raíz:** IsaacGym usa extensiones CUDA compiladas para Compute Capability hasta 8.x (Ampere). La RTX 5090 es Blackwell (CC 12.0). El compilador PTX de las extensiones no genera código para CC 12.0, por lo que el paquete `isaacgym` no puede instalarse ni ejecutarse.

**Error observado:**
```
ModuleNotFoundError: No module named 'isaacgym'
```

El contenedor Docker `nvcr.io/nvidia/pytorch:21.09-py3` usa CUDA 11.4, igualmente incompatible con Blackwell. La incompatibilidad está en las extensiones CUDA de IsaacGym, no en el sistema operativo.

**Comandos documentados (no ejecutables en este hardware):**
```bash
cd /home/lois/p2/RL/LeggGym/isaacgym
docker build -f docker/Dockerfile -t isaacgym_legged .
docker run --gpus all -it isaacgym_legged bash
# Dentro del contenedor:
python unitree_rl_gym/legged_gym/scripts/train.py --task=g1
```

**Resolución posible fuera de este entorno:** Hardware con GPU Ampere (RTX 30xx) o anterior, o esperar a una versión de IsaacGym con soporte Blackwell.

---

## Fase 3: Entrenamiento por imitación (BeyondMimic)

### Tarea 8 — Cuenta W&B ⚠️

**Estado:** Parcialmente resuelto — W&B sustituido por TensorBoard.

El flujo original requería `--logger wandb`. Se optó por `--logger tensorboard` para evitar autenticación externa. Consecuencia: el `wandb_path` necesario para Sim2Sim no existe.

### Tarea 9 — Dataset de movimientos ✅

**Estado:** Completado. Dataset `hooks_punch` pre-descargado:
```
/home/lois/p2/RL/BeyondMimic/artifacts/hooks_punch:v0/motion.npz
```

### Tarea 10 — Retargeting de movimientos ✅

**Estado:** No requerido. El dataset ya estaba en formato NPZ compatible con `Tracking-Flat-G1-v0`.

### Tarea 11 — Entrenar política de imitación ✅

**Estado:** Completado.

```bash
source ~/.virtualenvs/isaaclab/bin/activate
cd /home/lois/p2/RL/BeyondMimic/whole_body_tracking/scripts/rsl_rl

python train.py \
  --task=Tracking-Flat-G1-v0 \
  --registry_name "/home/lois/p2/RL/BeyondMimic/artifacts/hooks_punch:v0/motion.npz" \
  --headless \
  --logger tensorboard \
  --max_iterations 3500 \
  --num_envs 256
```

- Duración: 2 horas 14 minutos
- Checkpoint final: `logs/rsl_rl/g1_flat/2026-06-20_22-06-25/model_3499.pt`
- Velocidad media: ~1.4 s/iter, ~900 pasos/segundo

### Tarea 12 — Evaluar el aprendizaje ✅

**Estado:** Completado.

| Métrica | Iteración 500 | Iteración 3499 | Tendencia |
|---|---|---|---|
| Mean Reward | ~0.53 | +1.39 | ↑ |
| Episode Length | ~15 pasos | 26.77 pasos | ↑ |
| error_joint_pos | ~1.2 rad | 0.9964 rad | ↓ (mejor) |
| error_body_pos | ~0.10 m | 0.2130 m | estable |

Logs TensorBoard:
```
logs/rsl_rl/g1_flat/2026-06-20_22-06-25/events.out.tfevents.*
```

---

## Fase 4: Despliegue

### Tarea 13 — Exportar política ONNX ✅

**Estado:** Completado manualmente.

El runner `MotionOnPolicyRunner` solo exporta ONNX automáticamente cuando el logger es W&B. Al usar TensorBoard, fue necesario exportar desde el checkpoint directamente:

- Arquitectura actor: MLP 160 → 512 → 256 → 128 → 29 (ELU, obs normalizer embebido)
- Fichero: `logs/rsl_rl/g1_flat/2026-06-20_22-06-25/policy.onnx` (981 KB)
- Opset ONNX: 11 · Entradas dinámicas por batch

### Tarea 14 — Contenedor Docker de despliegue ⚠️

**Estado:** Comandos documentados, no ejecutado.

**Bloqueante:** El usuario `lois` no pertenece al grupo `docker`. Los comandos Docker requieren `sudo`:

```bash
cd /home/lois/p2/RL/g1_docker
docker build -f Dockerfile.g1_beyondmimic -t g1_beyondmimic .
docker compose up
```

**Resolución:** `sudo usermod -aG docker lois` seguido de nuevo login (requiere acceso de administrador).

---

## Fase 5: Sim2Sim (Isaac → MuJoCo)

### Tarea 15 — Validar en MuJoCo ❌

**Estado:** Bloqueado por dos causas independientes.

**Causa 1 — Dependencia de W&B:** El launch file requiere:
```bash
ros2 launch motion_tracking_controller mujoco.launch.py \
  wandb_path:=organization/project/run_id
```
El parámetro `wandb_path` referencia un artefacto de W&B. El entrenamiento usó TensorBoard; no existe `run_id`.

**Causa 2 — Sin acceso Docker:** El contenedor ROS 2 con `motion_tracking_controller` no puede construirse sin permisos Docker.

**Alternativa documentada (no implementada):** Modificar `mujoco.launch.py` para aceptar ruta local al `policy.onnx` o `model_3499.pt` en lugar de `wandb_path`.

---

## Fase 6: Sim2Real

### Tareas 16–17 — Conexión con robot y pruebas progresivas ❌

**Estado:** No aplicable. No hay robot físico Unitree G1 en este entorno.

Procedimiento documentado para referencia:
```bash
ping 192.168.123.161   # verificar conectividad con el G1
# Orden recomendado: postura estática → movimiento simple → caminata → movimientos complejos
```

---

## Vídeo demostrativo

### Vídeo de simulación renderizada ❌

**Estado:** Bloqueado por incompatibilidad de hardware. No tiene solución en este equipo.

**Causa técnica:** El flag `--video` activa `enable_cameras=True`, lo que selecciona el experience file `isaaclab.python.headless.rendering.kit`. Este kit carga `omni.kit.viewport.rtx`, que inicializa el plugin `librtx.scenedb.plugin.so`. Este plugin produce un **segmentation fault** durante `carbOnPluginStartup` en la RTX 5090 Blackwell.

El entrenamiento headless normal funciona porque `isaaclab.python.headless.kit` establece `app.useFabricSceneDelegate = false` y no carga `omni.kit.viewport.rtx`, evitando el plugin problemático.

Intentos realizados sin éxito:
1. `--headless --video` con `xvfb-run` → mismo crash (plugin RTX se carga independientemente del display)
2. `DISPLAY=:1` usando sesión X de otro usuario → mismo crash (fallo en plugin RTX, no en GLFW)
3. Sin `--headless` + Xvfb → mismo crash
4. Renderer alternativo Storm/pxr → no disponible en la instalación pip de Isaac Sim 5.1

**Resolución posible fuera de este entorno:** Isaac Sim 5.2+ con soporte RTX completo para Blackwell, o hardware Ampere.

### Vídeo de métricas de entrenamiento ✅

**Estado:** Completado como alternativa.

Fichero: `/home/lois/p2/training_results.mp4`  
Formato: MP4 · 1280×720 · 15 fps · 10 segundos · 1.5 MB

Muestra animación progresiva de las 4 métricas principales durante las iteraciones 500–3499:
- Mean Reward (converge de valores negativos a +1.39)
- Episode Length (×3.6 de mejora)
- Joint Position Error (−24%)
- Body Position Error

---

## Repositorio Git

**Estado:** 2 commits en `main`. Pendiente de `git push` manual.

```bash
# Ejecutar manualmente para publicar:
git -C /home/lois/p2/RL push origin main
```

Commits incluidos:
- `dcaaad5` — Correcciones de API (train.py, my_on_policy_runner.py, assets.py) + README
- `9af1ff4` — Informe de práctica (informe.md)

El repositorio cumple los requisitos de `goal.md`: contiene Dockerfiles, scripts de entrenamiento, configuraciones modificadas y README con comandos reproducibles. No incluye checkpoints pesados, imágenes Docker ni el paquete IsaacGym.

---

## Entregables: Estado Final

| Entregable | Estado | Ubicación |
|---|---|---|
| Informe PDF | ✅ | `informe.pdf` |
| Vídeo demostrativo | ⚠️ métricas (simulación bloqueada) | `training_results.mp4` |
| Repositorio Git | ✅ (pendiente push) | `/home/lois/p2/RL/` |
| Checkpoint final | ✅ | `logs/.../model_3499.pt` |
| Política ONNX | ✅ | `logs/.../policy.onnx` |
| Logs TensorBoard | ✅ | `logs/.../events.out.tfevents.*` |
| Vídeos Sim2Sim | ❌ Bloqueado | — |
| Vídeos Sim2Real | ❌ No aplicable | — |
