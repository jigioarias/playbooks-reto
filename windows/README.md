# Playbook de Mantenimiento Windows

Automatiza 4 tareas sobre servidores Windows vía WinRM y genera un informe final:

1. Valida el espacio disponible en `C:\`.
2. Verifica el estado de servicios críticos (por defecto: `wuauserv`, `W32Time`, `Spooler`, `Dnscache`).
3. Healthcheck de la VM: uso de CPU, memoria libre, uptime y reinicio pendiente.
4. Actualiza el sistema operativo vía Windows Update (`ansible.windows.win_updates`).
5. Genera un informe (`.txt`) por servidor, tanto en el host remoto (`C:\Temp\`) como descargado al nodo control en `./reportes_windows/`.

## Estructura

```
ansible_windows/
├── playbook.yml
├── inventory/
│   └── hosts.ini
├── templates/
│   └── report.j2
└── README.md
```

## Requisitos previos

### En el nodo control (desde donde ejecutas Ansible, típicamente Linux/macOS)

```bash
pip install "pywinrm>=0.4.0"
ansible-galaxy collection install ansible.windows
```

### En cada servidor Windows destino

- WinRM habilitado y configurado (se recomienda HTTPS/5986).
- PowerShell 5.1+ y .NET Framework 4.x.
- Usuario con privilegios administrativos.
- Para habilitar WinRM rápidamente en un servidor nuevo, Microsoft provee el
  script `ConfigureRemotingForAnsible.ps1` (buscar en la documentación oficial
  de Ansible para Windows).

## Configuración

1. Edita `inventory/hosts.ini` con tus servidores reales.
   - **No dejes contraseñas en texto plano**: usa `ansible-vault` para cifrar
     la variable `ansible_password` (el archivo de ejemplo ya referencia
     `vault_windows_password` como placeholder).
2. Ajusta variables en `playbook.yml` según tu entorno:
   - `critical_services`: lista real de servicios a monitorear.
   - `min_disk_free_percent`: umbral de alerta de disco.
   - `max_cpu_load_percent` / `min_memory_free_percent`: umbrales del healthcheck.
   - `update_categories`: categorías de Windows Update a instalar.

## Ejecución

```bash
# Prueba de conectividad
ansible -i inventory/hosts.ini windows_servers -m ansible.windows.win_ping

# Ejecución real
ansible-playbook -i inventory/hosts.ini playbook.yml

# Con vault para la contraseña
ansible-playbook -i inventory/hosts.ini playbook.yml --ask-vault-pass

# Solo contra un host puntual
ansible-playbook -i inventory/hosts.ini playbook.yml --limit winserver1.midominio.com
```

## Salida esperada

- En consola: resumen final por host (disco, servicios, healthcheck, actualización).
- En `./reportes_windows/informe_<host>.txt`: informe completo y detallado.
- En el propio servidor: copia en `C:\Temp\informe_mantenimiento_<host>.txt`.

## Notas importantes

- `win_updates` **no reinicia automáticamente** el servidor (`reboot: false`);
  el informe indica si queda un reinicio pendiente para que lo gestiones en
  una ventana de mantenimiento controlada (puedes usar el módulo
  `ansible.windows.win_reboot` en un play aparte si decides automatizarlo).
- El módulo `win_updates` puede tardar varios minutos dependiendo de la
  cantidad de actualizaciones pendientes; considera aumentar el timeout
  (`ansible_command_timeout`) en entornos con muchas actualizaciones acumuladas.
- Ajusta `ansible_winrm_transport` y la validación de certificados según la
  política de seguridad de tu organización; `server_cert_validation: ignore`
  es solo para pruebas.
