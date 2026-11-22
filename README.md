## Objectif multimodal

Le projet collecte des publications combinant texte et image afin d’étudier la désinformation sous plusieurs angles. Le pipeline conserve séparément la véracité du texte, l’authenticité de l’image, son contexte et la cohérence texte-image.

Une publication peut donc être trompeuse même si son texte est correct : l’image peut être manipulée, générée par IA ou sortie de son contexte. Les valeurs d’annotation prévues sont notamment :

```text
text_label: real | fake | unknown
image_label: authentic | manipulated | ai_generated | unknown
context_label: relevant | out_of_context | unknown
text_image_relation: coherent | incoherent | unknown
global_label: reliable | misleading_multimodal | fake | unknown
```
