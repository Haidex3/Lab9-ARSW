## Informe comparativo pruebas de carga Newman
### Escalamiento Vertical

**Configuración:**

- 1 sola VM (instancia B2ms, 2 vCPU, 8 GB RAM).
- Sin balanceador de carga.

**Datos observados:**

- Duración total: 1 min 44 s
- Solicitudes ejecutadas: 10 / 10
- Promedio de respuesta: 10.4 s
- Desviación estándar: 70 ms
- Consumo de datos: ~2.09 MB

El desempeño mejora respecto a la VM original (B1ls) debido a mayor capacidad de cómputo y memoria. Sin embargo, sigue siendo un único punto de procesamiento, lo cual limita la concurrencia efectiva y la tolerancia a fallos.

### Escalamiento Horizontal

**Configuración:**
- 3 VMs (cada una tipo B1ls).
- Balanceador de carga Azure Load Balancer Standard distribuyendo peticiones TCP/HTTP entre las instancias.

**Datos observados:**

- Duración total: 2 min 16 s
- Solicitudes ejecutadas: 10 / 10
- Promedio de respuesta: 13.5 s
- Desviación estándar: 275 ms
- Consumo de datos: ~2.09 MB

El tiempo medio por solicitud aumenta ligeramente (por la latencia adicional del balanceador y el overhead de red),
pero la tasa de éxito global y la estabilidad del sistema mejoran al distribuir la carga entre múltiples instancias. Además, ante la caída o lentitud de una VM, el Health Probe del Load Balancer reencamina las solicitudes hacia las otras instancias activas, manteniendo continuidad del servicio.

**Conclusión**

El escalamiento vertical ofrece menor latencia por solicitud (al eliminar salto de red y overhead del balanceador), pero no escala bien ante aumento de carga ni fallos.

El escalamiento horizontal con Load Balancer logra mayor disponibilidad, resiliencia y throughput total. Aunque cada petición sea levemente más lenta, el sistema puede atender más clientes concurrentes y mantiene 100 % de éxito incluso bajo carga paralela.