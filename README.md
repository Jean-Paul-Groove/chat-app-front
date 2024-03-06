# Chat-app

Application de chat en direct utilisant le realtime engine de Kuzzle

##  Features

Permet d'envoyer des messages jusqu'à 255 caractères

Détecte un mot impoli et avertit les usagers

## Prérequis
Avoir une instance de kuzzle active en local

## Notification reçues
Les notifications sont de format 

`{
  "type": "document" |"user"|"server",
  "status": number,
  "action": string,
  "scope": "in" |"out",
  "result": {
    "_id": string,
    "_source": KDocumentContent
    "_version": number
  },
  "node": string,
  "event": "write" | "delete" | "publish",
  "requestId": string,
  "timestamp": number,
  "volatile": JSONObject,
  "index": string,
  "collection": string,
  "controller": string,
  "protocol": "websocket",
  "room": string
}
`
Dans ce projet les notifications qui nous intéressent sont de type "document" et action "create".