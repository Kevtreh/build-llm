# build-llm

Agent [OpenCode](https://opencode.ai) qui combine un agent de code type **Build** et le
**délestage de contexte** vers un petit LLM local (via [LM Studio](https://lmstudio.ai) ou
tout serveur compatible OpenAI).

Quand une tâche serait trop lourde en contexte (lire de gros fichiers, parcourir beaucoup de
fichiers, analyser de longs logs, résumer une large zone de code), l'agent délègue la lecture
au sous-agent local `locallm` et ne récupère que le résumé. Le contenu brut ne pollue donc
jamais le contexte du modèle principal.

## Contenu

| Fichier | Rôle |
| --- | --- |
| `agents/build-llm.md` | L'agent primaire `build-llm` (style Build + délégation). |
| `opencode.example.jsonc` | Extrait de config : provider `local` + sous-agent `locallm`. |
| `LICENSE` | MIT. |

## Prérequis

- OpenCode **v2**.
- Un serveur compatible OpenAI local, par exemple LM Studio, exposant `http://localhost:1235/v1`.
- Un modèle chargé dans ce serveur (ex. `qwen3.5-2b-uncensored-hauhaucs-aggressive`).

## Installation

1. Copie l'agent dans ta config globale OpenCode :

   ```powershell
   Copy-Item .\agents\build-llm.md "$env:USERPROFILE\.config\opencode\agents\build-llm.md"
   ```

2. Fusionne le contenu de `opencode.example.jsonc` dans
   `~/.config/opencode/opencode.jsonc` (crée-le s'il n'existe pas).

3. Adapte si besoin `settings.baseURL`, le `modelID` et `limit.context` (voir ci-dessous).

4. Dans OpenCode, sélectionne l'agent **`build-llm`** (ou définis-le via `default_agent`).

## Configuration

Le provider `local` pointe vers ton serveur OpenAI-compatible :

```jsonc
"providers": {
  "local": {
    "name": "Local (LM Studio :1235)",
    "package": "@opencode/ai/providers/openai-compatible",
    "settings": {
      "baseURL": "http://localhost:1235/v1",
      "apiKey": "lm-studio"
    },
    "models": {
      "qwen3.5-2b": {
        "modelID": "qwen3.5-2b-uncensored-hauhaucs-aggressive",
        "name": "Qwen3.5 2B Uncensored (local)",
        "capabilities": { "tools": true, "input": ["text", "image"], "output": ["text"] },
        "limit": { "context": 17408, "output": 8192 }
      }
    }
  }
}
```

Le sous-agent `locallm` est **en lecture seule** et bridé en étapes pour éviter les boucles :

```jsonc
"locallm": {
  "mode": "subagent",
  "model": "local/qwen3.5-2b",
  "steps": 6,
  "permissions": [
    { "action": "edit", "resource": "*", "effect": "deny" },
    { "action": "shell", "resource": "*", "effect": "deny" }
  ]
}
```

### Personnalisation

- **Autre port / autre machine** : change `settings.baseURL`
  (ex. `http://gpu-host:1234/v1`).
- **Autre modèle** : change `modelID` (l'identifiant renvoyé par `/v1/models`).
- **Plus de contexte** : aligne `limit.context` sur la valeur réellement chargée dans ton
  serveur (sinon troncature ou erreurs).
- **Autre nom de sous-agent** : si tu renommes `locallm`, mets à jour le prompt de
  `agents/build-llm.md` et la permission `subagent:locallm`.

## Notes

- `locallm` est un petit modèle : idéal pour de la lecture/résumé, pas pour du raisonnement
  complexe. Pour les cas difficiles, préfère les sous-agents `explore` / `general`.
- Tout sous-agent (y compris `explore`/`general`) décharge déjà le contexte ; `locallm` ajoute
  l'avantage d'être local et gratuit.
- Un `system` d'agent non vide **remplace** le prompt de base : garde les instructions
  essentielles dans le corps Markdown.

## Licence

MIT — voir [LICENSE](LICENSE).
