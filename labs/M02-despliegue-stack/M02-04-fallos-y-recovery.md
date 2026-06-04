# Laboratorio M02-04 — Operación: romper, diagnosticar, recuperar

[← Página anterior](M02-03-filebeat-ingesta-viva.md) · [▲ Módulo M02](README.md) · [Siguiente →](M02-05-ha-shards-replicas.md)

> ⏱️ ~40 min · 🧩 Requisitos: M02-03 (stack completo) · 🖥️ Terminal + Discover

Simulamos fallos reales — ES caído, Beat caído, Discover vacío —, documentamos síntoma → comprobación → acción y recuperamos el stack con el ritual de M01-04.

---

### Paso 1 — Estado sano de referencia

M02-04 es un **runbook en vivo**. Cada escenario parte del mismo baseline (`health-check` + `_count`) para cuantificar recovery — «¿volvió a subir el contador?» es la pregunta que cierra cada incidente simulado.

```bash
./scripts/health-check.sh
C0=$(curl -fsS 'http://localhost:9200/filebeat-*/_count' | grep -o '"count":[0-9]*' | cut -d: -f2)
echo "baseline count=$C0"
```

---

### Paso 2 — Escenario A: Elasticsearch caído

Cuando ES no escucha, Filebeat **reintenta** (veremos `connection refused` en logs) y bufferiza en disco hasta cierto límite. Tras el `start`, no hace falta reinstalar el Beat — la ingesta se reanuda sola. Eso es distinto de un mapping error, que sí puede dropear eventos permanentemente.

```bash
docker compose -f infra/docker-compose.yml stop elasticsearch
docker logs lab-filebeat --tail 15
curl -fsS http://localhost:9200/ 2>&1 | head -1
```

Salida esperada en Filebeat: `connection refused` o error de conexión a `elasticsearch:9200`.

Repara:

```bash
docker compose -f infra/docker-compose.yml start elasticsearch
# espera healthy en: docker compose ps elasticsearch
sleep 20
docker logs lab-filebeat --tail 5
C1=$(curl -fsS 'http://localhost:9200/filebeat-*/_count' | grep -o '"count":[0-9]*' | cut -d: -f2)
echo "count tras recovery=$C1"
```

`C1` debe ser > `C0`.

---

### Paso 3 — Escenario B: Discover vacío (checklist real)

El síntoma más reportado en soporte — «Kibana no muestra nada» — suele ser **rango temporal** o data view incorrecto, no clúster caído. Este checklist nos obliga a ir de ES hacia la UI, no al revés.

Simula olvido del operador: time picker estrecho.

1. En Discover (`filebeat-*`), pon el rango a **Last 15 minutes** si hay pocos eventos.
2. Si no vemos nada, ejecutamos en terminal:

```bash
curl -fsS 'http://localhost:9200/filebeat-*/_count'
docker compose -f infra/docker-compose.yml ps filebeat
```

3. Si `_count` > 0 y Filebeat `Up` → amplía time picker a **Last 24 hours**.

Completamos el runbook (ejemplo de fila completa — adaptado al lab):

| Síntoma | Comprobación 1 | Comprobación 2 | Acción |
|---------|----------------|----------------|--------|
| Discover vacío | `_count` ¿crece al repetir curl? | ¿`lab-filebeat` Up en `docker compose ps`? | Si count > 0 y Beat Up → ampliar time picker; si count = 0 → logs de Filebeat y perfil `beats` |.

Compara con [TROUBLESHOOTING](../TROUBLESHOOTING.md).

---

### Paso 4 — Escenario C: red interna vs localhost

Docker Compose crea una **red privada** con DNS por nombre de servicio. `localhost` dentro de un contenedor apunta al propio contenedor — error clásico al copiar URLs del host a configs de Beats. En prod el equivalente es confundir IP del nodo con VIP del load balancer.

```bash
docker exec lab-filebeat sh -c 'wget -qO- http://elasticsearch:9200/ 2>/dev/null | head -c 80' || \
  docker exec lab-filebeat sh -c 'curl -fsS http://elasticsearch:9200/ | head -c 80'
docker exec lab-filebeat sh -c 'curl -fsS http://localhost:9200/ 2>&1 | head -1' || true
```

Dentro del contenedor **`elasticsearch:9200` funciona**; `localhost:9200` no apunta al nodo ES.

---

### Paso 5 — Recovery completo cronometrado

Cierra el módulo repitiendo el ritual de M01-04, pero ahora sabiendo **qué hace cada capa** que levantamos. Anotamos el tiempo total — en M12 compararás recovery con sizing de JVM y shards.

```bash
docker compose -f infra/docker-compose.yml --profile beats down
date +%H:%M:%S
docker compose -f infra/docker-compose.yml --profile beats up -d
./scripts/health-check.sh
date +%H:%M:%S
```

Anotamos el tiempo total. Abrimos Discover y confirmamos eventos `demo-app` en los últimos 15 min.

---

## Validación

- [ ] Filebeat reconectó tras caída de ES sin reinstalar.
- [ ] Tenemos runbook de 3 líneas para “Discover vacío”.
- [ ] Entendemos por qué `localhost` falla dentro del contenedor Beat.
- [ ] Completamos un `down` / `up` con health-check OK.

---

## Antes de seguir

- Orden de arranque: ES → Kibana → Beats.
- Logs del contenedor antes de `pull` o reinstalar imágenes.
- `down -v` borra datos; en lab solo si quieres reset total.

### Reto (tómate tu tiempo)

1. `curl -fsS 'http://localhost:9200/_cat/shards?v' | grep UNASSIGNED` — ¿cuándo lo usarías? (Amplía en [M02-05](M02-05-ha-shards-replicas.md).)
2. Para `lab-filebeat` 2 min: ¿sube `_count`? ¿sigue green el cluster?
3. (Opcional) Baja `ES_JAVA_OPTS` y mide tiempo hasta `healthy` tras reinicio.

<details>
<summary>Ver respuestas</summary>

**1. Shards UNASSIGNED**

Cuando el clúster está **`red`** o sospechas shards primarios/replicas sin asignar tras caída de nodo, disco lleno o corrupción. Complementa con `_cluster/allocation/explain`.

**2. Filebeat parado 2 min**

- **`_count`:** no sube (o se congela).
- **Cluster:** puede seguir **green/yellow** — la salud del clúster no depende de que Filebeat ingiera.

**3. JVM más baja (opcional)**

Con `-Xms512m -Xmx512m` el arranque suele **tardar más** o fallar en hosts justos de RAM; cronometra desde `up` hasta `healthy` en el healthcheck y compara con 768m.

</details>
