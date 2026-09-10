# Kubernetes GitOps Platform
Ce repository contient l'infrastructure Kubernetes gérée en **GitOps via Argo CD (IAC)**.

## Install Kubernetes
```sh
curl -sfL https://get.k3s.io | sh -
```

## Install ArgoCD
```sh
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w
# Tous les pods status en "1/1 Running"

# Activer le mode insecure (ArgoCD derrière un reverse proxy TLS)
kubectl -n argocd patch cm argocd-cmd-params-cm -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server

# Recuperation du MDP compte "admin" (il sera supprimé par la suite)
#printf "%s\n" "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
```

## API speech-to-text et text-to-speech

L'application `speech` déploie [Speaches](https://speaches.ai/) dans le namespace
`speech`. Elle expose une API compatible OpenAI via `https://speech.siluni.fr`.
Les modèles sont téléchargés automatiquement au premier appel et conservés dans
le PVC `speech-models` de 20 GiB.

Après le push Git et la synchronisation Argo CD :

```sh
kubectl -n speech get pods,svc,ingress
kubectl -n speech logs deploy/speech -f
```

### Speech-to-text

Le modèle STT est chargé à la demande. Cet exemple utilise `Systran/faster-whisper-small` :

```sh
curl -X POST https://speech.siluni.fr/v1/audio/transcriptions \
	-F file=@./audio.wav \
	-F model=Systran/faster-whisper-small \
	-F language=fr
```

Pour obtenir uniquement le texte, ajoutez `-F response_format=text`. Pour une
réponse JSON détaillée, utilisez `-F response_format=verbose_json`.

### Text-to-speech

```sh
curl -X POST https://speech.siluni.fr/v1/audio/speech \
	-H 'Content-Type: application/json' \
	-d '{"model":"speaches-ai/Kokoro-82M-v1.0-ONNX","input":"Bonjour depuis Kubernetes.","voice":"af_heart","response_format":"mp3"}' \
	--output ./bonjour.mp3
```

Les modèles déjà téléchargés peuvent être inspectés avec :

```sh
curl https://speech.siluni.fr/v1/models
```

Le catalogue des modèles disponibles est accessible avec :

```sh
curl 'https://speech.siluni.fr/v1/registry?task=automatic-speech-recognition'
curl 'https://speech.siluni.fr/v1/registry?task=text-to-speech'
```

Pour une application Python, installez `openai` puis utilisez `base_url` :

```python
from openai import OpenAI

client = OpenAI(base_url="https://speech.siluni.fr/v1", api_key="not-used")

with open("audio.wav", "rb") as audio:
		result = client.audio.transcriptions.create(
				model="Systran/faster-whisper-small", file=audio, language="fr"
		)
print(result.text)

speech = client.audio.speech.create(
		model="speaches-ai/Kokoro-82M-v1.0-ONNX",
		voice="af_heart",
		input="Bonjour depuis Kubernetes.",
)
speech.stream_to_file("bonjour.mp3")
```

Le premier téléchargement peut prendre plusieurs minutes et consommer plusieurs
Go de disque et de mémoire. L'Ingress est volontairement simple comme les autres
applications du dépôt : configurez une clé API Speaches via un Secret Kubernetes
(`API_KEY`) ou une authentification Traefik avant de rendre ce service accessible
publiquement.

### Ajouter la clef privee SOPS (decode secret)
```sh
kubectl -n argocd create secret generic sops-age --from-literal=keys.txt="AGE-SECRET-KEY-1XXXX..."
```

### Bootstrap sync ArgosCD
```sh
kubectl apply -n argocd -f bootstrap/root-application.yaml
```
