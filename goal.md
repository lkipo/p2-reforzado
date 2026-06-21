Cada alumno entregará un .zip con:

    Informe breve en PDF.
    Enlace a un vídeo demostrativo, 3-6 minutos.
    Enlace al repositorio de trabajo.
    Resultados generados relevantes (opcional): logs, checkpoints pequeños o enlace externo si son pesados.

El informe debe incluir:

    Equipo utilizado: sistema operativo, GPU, memoria de GPU y driver NVIDIA.
    Comandos usados para construir y ejecutar los contenedores.
    Resultado de la parte LeggedGym:
        test del entorno,
        entrenamiento de la tarea g1,
        logs o checkpoint generado.
    Resultado de la parte BeyondMimic:
        preparación del entorno,
        entrenamiento o ejecución de ejemplo,
        Sim2Sim si se alcanza esa fase.
    Problemas encontrados y soluciones aplicadas.
    Breve conclusión sobre lo conseguido.

El vídeo debe mostrar:

    Arranque del entorno.
    Ejecución de la parte LeggedGym:
        test inicial,
        entrenamiento corto,
        logs o resultados.
    Ejecución de la parte BeyondMimic:
        entorno preparado,
        comando principal ejecutado,
        resultado obtenido.
    Comentario breve de los problemas encontrados.

Cada alumno debe entregar un enlace a un repositorio Git, preferiblemente privado compartido con el profesor, que contenga:

    Los archivos Docker utilizados.
    Los scripts de ejecución.
    Las configuraciones modificadas de LeggedGym y BeyondMimic.
    Un README.md con los comandos necesarios para reproducir la práctica.
    No debe incluir archivos pesados como imágenes Docker, datasets grandes, checkpoints grandes o el paquete de Isaac Gym.

No se exige que las políticas entrenadas sean óptimas. Se evaluará principalmente que el entorno funcione, que los comandos sean reproducibles y que se documente claramente lo realizado.