## Dependencies

Dependencies are defined in containers.

- Use container references that resolve correctly across container engines:
    - `docker://image:tag` is a Singularity/Apptainer-style reference and breaks under plain Docker (`docker: invalid reference format`).
    - Always include the full registry host in the reference (e.g. `quay.io/biocontainers/...`, not just `biocontainers/...`). Omitting it makes the engine default to Docker Hub. 
    - Document which container engines are officially supported (Docker, Singularity, Apptainer, Conda). 
    