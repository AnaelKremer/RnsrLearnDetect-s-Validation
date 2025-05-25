# Processus destiné à faciliter la détection d'affiliations en consolidant les résultats du web service rnsrLearnDetect.

A partir des adresses d'un corpus, on utilise le web service rnsrLearnDetect. On interroge ensuite le web service Loterre Expand pour enrichir les données relatives aux rnsr trouvés.

On obtient alors des informations comme le sigle du laboratoire, son code Unité, son intitulé, des noms alternatifs etc. 

On réalise ensuite une série de tests sur les adresses originales pour déterminer s'il y a :

- Présence du code CNRS
- Présence du sigle du laboratoire
- Présence de l'intitulé exact du laboratoire
- Présence de noms alternatifs (souvent des codes unités non CNRS)
- Présence de mots de l'intitulé du laboratoire (ex : "volcans" et "magma" sont présents dans "lab volcans & magma")
- Présence d'abréviations de l'intitulé du laboratoire (ex : "evo", "div" et "bio" sont présents dans "univ toulouse, umr evolut & divers biol, cnrs, france")

On génère enfin des chaînes pour verbaliser de façon synthétique les multiples résultats obtenus :

- **RNSR non détecté** siginifie que le web service rnsrLearnDetect n'a pas pu associer de RNSR à l'adresse interrogée
- **RNSR non répertorié dans Loterre** siginifie que le web service rnsrLearnDetect a détecté un RNSR mais que celui-ci n'est pas dans Loterre, aucun test n'est donc possible
- **Aucun match** aucun des tests n'a renvoyé de résultat positif
- dans tous les autres cas, on concatène dans une chaîne les différents résultats possibles, à savoir :
  - **Code CNRS détecté**
  - **Sigle détecté**
  - **Intitulé exact trouvé**
  - **x altLabel(s) trouvé(s)**
  - **x mot(s) trouvé(s)**
  - **x abréviation(s) trouvée(s)**

Cela permet à minima, dans le cadre d'un travail d'identification des affiliations d'un corpus, de ne pas avoir à vérifier que tous les rnsr trouvés sont bien conformes aux affiliations et de concentrer son attention sur les adresses n'ayant aucun match, ou très peu.
