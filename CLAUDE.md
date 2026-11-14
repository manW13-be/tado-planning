# CLAUDE.md — tado-planning

## PyTado — fork utilisé

**Ne pas** installer `python-tado` depuis PyPI ni modifier des attributs privés de PyTado.

Le Dockerfile et `tado-planning-run.py` utilisent **`manW13-be/python-tado`** — notre fork GitHub de `python-tado` 0.19.2, publié sur `https://github.com/manW13-be/python-tado`.

### Installation

```dockerfile
RUN pip install --no-cache-dir flask requests \
    "git+https://github.com/manW13-be/python-tado.git"
```

### Usage

```python
from PyTado.interface.interface import Tado

tado = Tado(token_file_path=TOKEN_FILE, client_id=client_id)
# Aucun patch ni monkey-patching nécessaire
```

### Corrections incluses dans le fork (vs upstream 0.19.2)

- `client_id` en paramètre du constructeur — plus de mutation de `PyTado.const.CLIENT_ID_DEVICE`
- URL du device flow inclut `client_id` — corrige le rejet `missing_client_id` de `login.tado.com`
- Refresh token envoyé en `application/x-www-form-urlencoded` (l'upstream envoie en query string)
- Token expiré → fallback automatique vers device flow dans le constructeur
- `_device_ready()` wrappé en try/except dans le constructeur — si `/me` échoue (rate-limit 429, réponse vide), le constructeur efface le token et démarre un device flow au lieu de crasher
- `_get_id()` essaie `homes[0].id`, `homeId`, `home.id`, `homeIds[0]` avant de lever une exception

### Point subtil — polling device flow vs refresh token

Le form-encoding (`application/x-www-form-urlencoded`) est correct pour le **refresh token** et le **device authorization** mais **pas** pour le **polling** du device code. L'endpoint de polling (`_check_device_activation`) attend des `params=` (query string), pas un form body. Si on uniformise en form-encoded partout, le polling casse silencieusement — l'utilisateur peut s'authentifier dans le navigateur mais le script ne le détecte jamais.

### Accès HA

SSH : `ssh admin@ha2.local`
Repo sur HA : `/config/tado-planning/`
Run hors docker : `ssh admin@ha2.local "cd /config/tado-planning && ./tado_planning/run.sh"`

### Sources locales

- `/Users/emmanuel/Developer/HomeWork/Domotic@VoetHome/tado-test/` — repo git source du fork (`manW13-be/python-tado`), branche `main`
- `/Users/emmanuel/Developer/HomeWork/Domotic@VoetHome/_dependencies/PyTado/` — clone de `manW13-be/PyTado` (fork de wmalgadey v1.0.0), branche `fix/device-flow-client-id-token-refresh` pour la PR #314 upstream — **ne pas installer depuis ce repo dans tado-planning**

### Statut upstream

- **PR #314** sur `wmalgadey/PyTado` — nos corrections portées sur la base v1.0.0 (Pydantic), en attente de merge
- **PR #1** sur `gazzer82/PyTado` — corrections similaires sur base Python 3.x, en attente
- Une fois PR #314 mergée et publiée sur PyPI, le Dockerfile pourra revenir à `pip install python-tado` sans fork

La lib async alternative est `tadoasync` (`erwindouna/python-tado`) — async/await, Python ≥3.12. Migration non prioritaire (nécessite réécriture async).
