# Processus destiné à faciliter la détection d'affiliations en consolidant les résultats du web service rnsrLearnDetect.

A partir des adresses d'un corpus, on utilise le web service rnsrLearnDetect. On interroge ensuite le web service Loterre pour enrichir les données relatives aux rnsr trouvés.

On obtient alors des informations comme le sigle du laboratoire et son code Unité. On teste ensuite si ces sigles et codes font effectivement partie des adresses interrogées.

On récupère comme résultat des valeurs du type "Code détecté & Sigle détecté".

Cela permet à minima, dans le cadre d'un travail d'identification des affiliations d'un corpus, de ne pas avoir à vérifier que tous les rnsr trouvés sont bien conformes aux affiliations.
