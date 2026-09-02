# cloudflared

Déploie un tunnel Cloudflare ([`cloudflared`](https://github.com/cloudflare/cloudflared))
dans un cluster Kubernetes, en manifests k8s bruts appliqués via
`kubernetes.core.k8s` (pas de chart Helm officiel pour ce cas).

## Ce que fait le rôle

- Crée le namespace `cloudflared`
- Crée un `Secret` avec les credentials du tunnel, rendu depuis
  `templates/credentials.json.j2`
- Crée une `ConfigMap` avec la config du tunnel, rendue depuis
  `templates/config.yaml.j2`
- Déploie le `Deployment` `cloudflared`

## Prérequis / points d'attention

- Nécessite un kubeconfig valide en `~/.kube/config` pointant vers le
  cluster cible.
- ⚠️ Ce rôle ne fournit **aucune valeur par défaut** pour l'identité du
  tunnel ni pour les routes — `cloudflared_tunnel_id`,
  `cloudflared_account_tag`, `cloudflared_tunnel_secret` et
  `cloudflared_ingress` doivent être fournis par le consommateur
  (typiquement via `group_vars`/`host_vars`, chiffrés avec `ansible-vault`
  pour les valeurs sensibles). Voir [`defaults/main.yml`](defaults/main.yml).
- `cloudflared_tunnel_secret` authentifie le tunnel auprès de Cloudflare —
  à vaulter, jamais à committer en clair.
- `cloudflared_image_tag` est figé (pas de `latest`) pour éviter qu'un pull
  silencieux change l'image au prochain rollout/reschedule. Vérifie les
  nouvelles versions sur
  [github.com/cloudflare/cloudflared/releases](https://github.com/cloudflare/cloudflared/releases).
- Pas de `handlers/` : les manifests sont appliqués tels quels, rien à
  redémarrer côté Ansible (un changement de ConfigMap/Secret ne redéclenche
  pas automatiquement le rollout du Deployment — à gérer manuellement ou via
  un hash de contenu en annotation si besoin).

## Variables principales

Voir [`defaults/main.yml`](defaults/main.yml) pour la liste complète.

| Variable                       | Défaut                     | Description                                    |
| --------------------------------- | ---------------------------- | -------------------------------------------------- |
| `cloudflared_namespace`          | `cloudflared`                | Namespace k8s cible                            |
| `cloudflared_replicas`           | `2`                           | Replicas du Deployment                         |
| `cloudflared_image_repository`   | `cloudflare/cloudflared`      | Image du conteneur                             |
| `cloudflared_image_tag`          | `2026.8.3`                    | Version figée — voir [releases](https://github.com/cloudflare/cloudflared/releases) |
| `cloudflared_tunnel_id`          | *(aucun, requis)*            | UUID du tunnel (`cloudflared tunnel create`)   |
| `cloudflared_account_tag`        | *(aucun, requis)*            | Account tag Cloudflare                         |
| `cloudflared_tunnel_secret`      | *(aucun, requis, sensible)*  | Secret du tunnel — à vaulter                   |
| `cloudflared_ingress`            | *(aucun, requis)*            | Liste de `{hostname, service}` pour le routing |

## Exemple

```yaml
- hosts: localhost
  connection: local
  roles:
    - cloudflared
  vars:
    cloudflared_tunnel_id: "{{ vault_cloudflared_tunnel_id }}"
    cloudflared_account_tag: "{{ vault_cloudflared_account_tag }}"
    cloudflared_tunnel_secret: "{{ vault_cloudflared_tunnel_secret }}"
    cloudflared_ingress:
      - hostname: auth.example.com
        service: http://authentik-server.authentik.svc.cluster.local:80
      - hostname: grafana.example.com
        service: http://monitoring-grafana.monitoring.svc.cluster.local:80
```
