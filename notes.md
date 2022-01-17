# Contexte 

le modèle de chiffrement actuel, basé sur les clé publiques *RSA* est très vulnérable aux attaques par 
*ordinateurs quaniques*. il est donc nécessaire de trouver un autre modèle car lagrande majorité des chriffrements utilisés aujourd'hui sont basés sur la méthode RSA.  
RSA se base sur la résolution de problèmes mathématiques très complexes, ne pouvant pas être résolus par des ordinateurs classiques en un temps polynomial. Un ordinateur quantique de *grande taille*, en revanche, serait capable de résoudre ces problèmes en temps résonables.  

*LAC* est une méthode utilisant toujours les clés publiques mais résistants aux attaques quantiques. *Lattice based*. Dit "*Ring Learning With Errors, RLWE*. Considéré *efficace energétiquement ET en terme de mémoire en standard NIST*.  
Clé relativement petite, et arithmétique simple. 
 *  code *BCH* qui permet d'utiliser des nombre polynomiaux avec un *coeff a un seul octet*
 *  Méthode remplacant les mult par des add/sub  : polynomes d'erreurs a *coeff ternaires*
Le but est ici de proposer une implémentation d'une architecture codesign HW/SW  de cryptographie LAC.  
Deux choses proposées :  
HW accel pour LAC et leurs bottlenecks (génération de nb polynomiaux, multiplication de ces derniers et *correction d'erreurs*) (128/256b)
Création d'une extension RISC-V LAC et intégration des accelzerateurs dans un core RISCV  
Choix RISCV : extensible, gratuit, *securisé*. Implémentation dans le pipeline directement pour diminuer les couts de communication bus.

Augmentation de la taille ( *importante ?*) : 32K LUT, 11K registres et 2 slice DSP (*c'est l'augmentation ou le total ?*)


Quelques implémentations sous x86, arm et RISCV mais pas de résultats de perf, et peu d'utilisation de codesign. Intégration faible a l'ISA


# LAC post quantique
## Le problème RLWE 
deux problèmes en un :
* problème de recherche : trouver _s_ dans l'équatrion _b = a.s + e_ en conaissant le polynome _a_ (public) et le résultat _b_.   
_s_ est le coefficient polynomial secret, et _e_ est le coefficient de correction d'erreur spécifique a _a_
* problème de décision : *disntinguer le couple (a, b) d'un "uniform sample"*

![](keygen_encrypt_decrypt.png)
## Etape 1 : génération de clés
Obtention de la clé publique _pk_ à partir d'un couple _(graine, b)_ et de la clé secrète _sk_ valant _s_
Utilisation de sha256 pour obtention d'un nombre pseudo aléatoire pour *étendre* une graine  
_s_ et _e_ sont obtenus à l'aide d'une distribution binomiale
<!-->TODO do a fuck ton more reseach <!-->

## Etape 2 : Chiffrement
<!-->TODO huh ? <!-->
Ajout de bruit _u_ via une instance RLWE ?
## Etape 3 : Déchiffrement
Utilisation de la clé privée _sk_ pour retrouver la chaine de caractère initiale en soustrayant le terme _u.s_ de _v_ afin de filtrer les plus gros termes parasites. Le reste est retiré à l'aide d'un *BCH decoder*


# Accélération HW
## problème de la ganération de polynomes
Utilisation d'un accelerateur SHA256. *Voire référence 7*.

## problème de la multiplication de polynomes

LAC est basé sur ce type d'opération, a la fois pour créer des "instances RLWE" au chiffrement, et au déchiffrement.
Principe : remplacer les multiplciations par des additions et soustractions, des opérations beaucoup plus simples à faire sur des polynomes
Les chercheurs ont utilisé un module faisant des additions par *convolution positive* qu'ils ont modifié pour supporter également la *convolution négative* 
étant donné que LAC utilise les deux méthodes. Cette modification a nécessité d'ajouter des MUX et de modifier la partie de controle du circuit 

En plus de ca, ils voulaient développer un algo logiciel permettant de passer de manière flexible de LAC 128 à LAC196-256, en découpant les multiplications de polynomes en deux parties. (chaque ploynome est découpé en deux parties égales, et la multiplication combine les parties de chaque polynome).
En revanche, cet algorithme imposant de réaliser des multiplications de polynomes, et étant donné que les tests préliminaires indiquaient que la méthode des additions implémentée était très efficace, cette solution n'a pas été gardée.
A la place, ils ont re utilisé leur matériel et utilisé un double découpage (4 parties de 256 pour une multiplication deux polynomes de 1024)

# Correction d'erreurs (décodeur BCH)
## 3 étapes
* Détéction et calcul de *syndromes*
* Calcul du polynome d'erreur avec l'algorithme Berlekamp-Massey
* Calcul de la cause de l'erreur avec l'algorithme de Chien, accéléré par du matériel car partie la plus lente du programme.

MULGF : chaque bit de A et confronté a tous les bits de B en séquence en commencant par le dernier bit.
        Effectue des décalages via registres avec une boucle de retour injectant le dernier bit du résultat dans le décaleur aux positions 0 et 4
        Fin en 9 cycles
Chien accel :
        MULGF est utilisé pour sa flexibilité tout en restant relativement econome en espace requis. 4 d'entre eux sont utilisés en // pour accelerer Chien
        La sortie des MUGF est redirigée a leur entrée d'avoir a remettre à jour les valeurs des registres et gagner du temps (on n'a alors besoin de charger des
        valeurs qu'une fois toutes les 4 itérations)

# RISCV

Utilisation d'un processeur RISCV 4 étages (*RISCY*, IF, ID, EX, WB) qui implémente les sets d'instructions I C et M 
Composants :
* __IF__ buffer *prefetch*
* __ID__ décodeur d'instructions
* __ID__ GPR : banc de registres  
* __EX__ ALU : unité arithmétique et logique
* __EX__ MULT : unité de multiplications
* __EX__ PQ-ALU : l'ALU post quantique développée par les chercheurs contenant les 4 accelerateurs décrits (Chien et MULGF, multiplicateur polynomial et MOD q)
* __EX__ LSU : l'unité load-store 

Ajout de 4 instructions via la modularité du RISCV :
* pq.mul_ter 
* pq.mul_chien
* pq.sha256
* pq.modq  
Opcode dédié sur instructions de type R : 0x77. Activation du bon module accelerateur via le champ func3.  
Utilisation des input/output buffers car opérandes trop grandes pour les registres (qui font que 32b)

# Résultats 
## Attaques dans le code BCH

Les aurtes travaux ne sont pas exactement temps constant ce qui peut etre exploité lors d'attaques

## Implémentation
Implémentation sur Ultrascale+ et compilation w/ compilateur officiel Berkley pour RISCV.
Perf 20% inférieure aux autres implémentation mais plus sécurisée et configurable. Surtout du au module sha256, au code de correction d'erreur et au re encodage 
Potentielle amélioration : passer de sha256 à Keccak -> requiers plus d'espace mais plus performant
Aucune utilisation de BRAM, moins de DSP mais plus de LUT et reg

# Conclusion 
LAC fonctionne mais présente des goulots d'étranglement limitant les performances de la méthode. Ce papier présente une implémentation accélérée par du matériel
limitant ces goulots d'étranglements. De plus, le jeu d'instruction RISCV dédié permet d'augmenter encore les perf de cette implémentation tout en étant modulaire

