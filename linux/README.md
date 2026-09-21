# Playbook de Mantenimiento RHEL

Automatiza 4 tareas sobre servidores RedHat Linux y genera un informe final:

1. Valida el espacio disponible en la partición raíz (`/`).
2. Verifica el estado de servicios críticos (por defecto: `sshd`, `firewalld`, `chronyd`).
3. Crea el usuario `ecuador`.
4. Actualiza el sistema operativo (`dnf update`).
5. Genera un informe (`.txt`) por servidor, tanto en el host remoto como descargado al nodo control, dentro de `./reportes/`.

## Estructura

```
ansible_rhel/
├── playbook.yml
├── inventory/
│   └── hosts.ini
├── templates/
│   └── report.j2
└── README.md
```

## Requisitos previos

- Ansible 2.14+ instalado en el nodo control.
- Acceso SSH con privilegios de `sudo` a los servidores destino.
- Python instalado en los hosts remotos (requisito estándar de Ansible).

## Configuración

1. Edita `inventory/hosts.ini` con tus servidores reales.
2. Ajusta las variables en `playbook.yml` si lo necesitas:
   - `critical_services`: lista de servicios a monitorear.
   - `min_disk_free_percent`: umbral de alerta de disco.
   - `new_user_groups`: grupos adicionales para el usuario `ecuador` (ej. `wheel` para sudo).
3. **Importante (RHEL 7 vs 8/9):** el playbook usa el módulo `dnf`. Si tus
   servidores son RHEL 7, cambia `ansible.builtin.dnf` por `ansible.builtin.yum`
   en la tarea "4. Actualizar todos los paquetes del sistema" (los parámetros son iguales).

## Ejecución

```bash
# Prueba de conectividad
ansible -i inventory/hosts.ini rhel_servers -m ping

# Modo simulación (no aplica cambios)
ansible-playbook -i inventory/hosts.ini playbook.yml --check

# Ejecución real
ansible-playbook -i inventory/hosts.ini playbook.yml

# Solo contra un host puntual
ansible-playbook -i inventory/hosts.ini playbook.yml --limit servidor1.midominio.com
```

## Salida esperada

- En consola: un resumen final por host (disco, servicios, usuario, actualización).
- En `./reportes/informe_<host>.txt`: el informe completo y detallado.
- En el propio servidor: copia en `/tmp/informe_mantenimiento_<host>.txt`.

## Notas de seguridad

- La actualización del sistema (`state: latest`) puede reiniciar servicios o
  requerir reinicio del servidor (kernel). El playbook detecta si hay un
  reinicio pendiente pero **no reinicia automáticamente**; hazlo de forma
  controlada según tu ventana de mantenimiento.
- Se recomienda ejecutar primero con `--check` en entornos productivos.
