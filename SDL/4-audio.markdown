---
layout: article
title: Programmer des jeux sous Linux avec SDL <br/>Partie IV
section: Linux Programming
description: Programmation audio
tags:
- linux
- programming
- SDL
seo:
  type: TechArticle
locale: fr_FR
published: Planète Linux numéro 6
---
## Programmation audio

Nous continuons notre découverte de la bibliothèque SDL avec, ce mois-ci, la partie sonore. Mais avant de passer à l'initiation proprement dite, profitons-en pour vous donner quelques nouvelles récentes concernant l'état de l'art de SDL depuis notre dernier article.

SDL a enfin atteint sa version stable 1.0 et une nouvelle version de développement 1.1 a été lancée en parallèle. Par rapport à la précédente version 0.11, assez peu de changements dans 1.0. On peut noter l'ajout d'une nouvelle gestion de l'affichage sous X11 qui permet d'obtenir un affichage plein écran avec les serveurs XFree86, sans qu'il soit besoin d'utiliser l'extension DGA (et donc d'avoir des privilèges particuliers).

La version de développement 1.1 apporte d'autres ajouts majeurs : une API pour la gestion des joysticks, des timers multithreadés (ajoutés à SDL pour le portage de *Myth II*) et surtout, la gestion de la bibliothèque *OpenGL* via SDL, ce qui ouvre enfin la porte à la 3D d'une manière propre et portable. Les principales lacunes de SDL par rapport à *DirectX* sont donc comblées (le manque d'API pour la 3D et les joysticks étaient cruciaux) et SDL est en train de devenir incontournable pour les programmeurs de jeux sous Linux.

### SDL Audio : aperçu général

La partie audio de SDL est particulièrement simple et permet un accès aux fonctions de base. L'implémentation de fonctionnalités plus avancées est laissée à la charge de l'utilisateur, l'objectif de SDL étant encore et toujours de fournir une couche d'abstraction simple et directe (*Simple Directmedia Layer*). Il est possible, notamment, de faire appel à la librairie annexe "*SDL Mixer*" que nous aborderons un peu plus loin dans cet article.  

 
### Initialisation de SDL

Sous Linux ainsi que sous la plupart des systèmes supportés par SDL, le sous-système audio utilise une thread séparée. Afin d'initialiser correctement la librairie, il faudra tout d'abord veiller à appeler la fonction `SDL_Init()` avec le flag **SDL_INIT_AUDIO** :

 {% highlight c %}
SDL_Init(SDL_INIT_VIDEO | SDL_INIT_AUDIO);
{% endhighlight %}
  
Ceci a pour effet de démarrer la thread chargée de la gestion audio. Pour ouvrir le périphérique proprement dit, il faut recourir à la fonction :

{% highlight c %}
SDL_OpenAudio(SDL_AudioSpec *desired, SDL_AudioSpec *obtained);
{% endhighlight %}

Cette fonction prend en argument deux structures `SDL_AudioSpec` qui permettent de décrire le format audio désiré. La structure pointée par `desired` doit être remplie de la manière suivante :

{% highlight c %}
typedef struct {
         int freq;
{% endhighlight %}
         
Indique la fréquence en échantillons par seconde (par exemple, 44100 pour une qualité CD).

{% highlight c %}
         Uint16 format;
{% endhighlight %}

Indique le format des échantillons, parmi les valeurs décrites ci-dessous :

* **AUDIO_U8** : Echantillons 8 bits non signés,
* **AUDIO_S8** : Echantillons 8 bits signés,
* **AUDIO_U16LSB** : Echantillons 16 bits non signés (non encore supportés, car peu répandus),
* **AUDIO_S16LSB** : Echantillons 16 bits signés (les plus répandus),
* **AUDIO_U16MSB, AUDIO_S16MSB** : idem que ci-dessus pour les architectures big endian (pratiquement tout ce qui n'est pas Intel, comme les PowerPC ou les Alpha).

{% highlight c %}
         Uint8  channels;
{% endhighlight %}

Le nombre de canaux : 1 pour un son mono, 2 pour la stéréo.

{% highlight c %}
         Uint8  silence;
{% endhighlight %}

Valeur de silence pour le tampon (attention, il est calculé par SDL, vous ne devez pas le remplir !).

{% highlight c %}
        Uint16 samples;
{% endhighlight %}

Taille du tampon audio ; il ne s'agit pas de la taille maximale des échantillons que vous pourrez charger par la suite, mais plutôt de la taille du tampon utilisé de manière interne. D'une façon générale, un grand tampon demandera moins de temps CPU mais introduira un décalage plus grand entre l'arrivée des échantillons et la sortie sur les haut-parleurs. D'un autre côté une taille de tampon trop petite peut entraîner des déformations du signal audio sur les machines peu puissantes, qui auraient du mal à suivre.

{% highlight c %}
         Uint32 size;
{% endhighlight %}

Taille en octets du tampon, calculée automatiquement par `SDL_OpenAudio()`.

{% highlight c %}
         void (*callback)(void *userdata, Uint8 *stream, int len);
{% endhighlight %}

L'adresse d'une fonction appelée automatiquement par SDL à chaque fois que le tampon est vide. Cette fonction est chargée de remplir à nouveau le tampon.

{% highlight c %}
         void  *userdata;
{% endhighlight %}

Cette valeur sera passée comme premier argument de la fonction callback décrite ci-dessus. Sa signification est complètement laissée à la charge de l'utilisateur.

{% highlight c %}
 } SDL_AudioSpec;
{% endhighlight %}

Il est possible que le format audio obtenu soit différent de celui demandé, notamment en cas de contraintes matérielles. C'est pourquoi il faut toujours vérifier le contenu de la structure pointée par `obtained` qui décrit les caractéristiques du format audio effectivement alloué. Il est également possible de forcer un format audio, en appelant la fonction avec un second argument `NULL`. Dans ce cas, SDL fera son possible pour émuler le format audio en fonction des contraintes matérielles, si cela s'avère nécessaire.

Il est important de retenir que l'architecture audio de SDL repose sur des fonctions callbacks. Il n'y a pas de fonction simple pour jouer un son dans l'API (bien que cela puisse être implanté relativement aisément). Au lieu de cela, il faut écrire une fonction chargée de fournir à la demande de SDL les échantillons du son, par tranche correspondant à la taille du tampon audio alloué (le champ `samples` de la structure `SDL_AudioSpec`)...

Pour permettre à l'utilisateur d'initialiser correctement toutes les structures avant que SDL ne se mette à appeler des fonctions callback, l'appel à `SDL_OpenAudio()` met initialement SDL en mode "pause". La restitution et donc le remplissage des tampons audio ne commencent que lorsque le programme appelle explicitement `SDL_PauseAudio(0)`, ce qui a pour effet de sortir du mode pause.

Il est possible d'utiliser les fonctions `SDL_LockAudio()` et `SDL_UnlockAudio()` pour garantir que la fonction callback n'est pas appelée dans la thread audio (exclusion mutuelle). Cela peut être utile si vous devez fournir des données audio depuis une thread séparée (i.e. en dehors de la fonction callback qui est toujours exécutée dans la thread audio de SDL)...

### Charger un échantillon

SDL fournit une fonction pour charger un fichier son au format WAV en mémoire. Actuellement, seuls les fichiers WAV de base sont supportés (données brutes ou au format compressé ADPCM).

{% highlight c %}
SDL_AudioSpec *SDL_LoadWAV(const char *file, SDL_AudioSpec *spec, Uint8 **audio_buf, Uint32 *audio_len);
{% endhighlight %}

Son utilisation est la suivante : `file` est le chemin d'accès du fichier à charger. On fournit l'adresse d'une structure `SDL_AudioSpec` qui sera remplie par la fonction avec les données décrivant le format du fichier chargé (cette adresse sera également retournée par la fonction). La fonction se charge d'allouer un tampon en mémoire et d'y charger le son. Les arguments `audio_buf` et `audio_len` sont retournés par SDL, afin d'indiquer la longueur et l'adresse dudit tampon. En cas d'erreur, la fonction renvoie `NULL` au lieu de `spec`. Le tampon alloué doit, par la suite, être explicitement libéré par l'utilisateur à l'aide de la fonction `SDL_FreeWAV()`. 

Reportez-vous au programme d'exemple accompagnant cet article, pour un modèle d'utilisation détaillé de cette fonction.

### Convertir des échantillons

L'un des aspects les plus puissants de la partie audio de SDL est la possibilité de convertir aisément des échantillons d'un format à l'autre, par l'intermédiaire de blocs de conversion audio. Concrètement, un bloc de conversion audio est un objet `SDL_AudioCVT`, dont la structure est la suivante :

{% highlight c %}
typedef struct SDL_AudioCVT {
         int needed;        /* Valeur non-nulle si la conversion est possible */
         Uint16 src_format; /* Format audio source */
         Uint16 dst_format; /* Format audio cible */
         double rate_incr;  /* Rapport de conversion pour la fréquence */
         Uint8 *buf;        /* Tampon contenant les données audio */
         int    len;        /* Longueur du tampon d'origine ci-dessus */
         int    len_cvt;    /* Longueur du tampon de destination */
         int    len_mult;   /* Coefficient: le tampon doit avoir une taille de len*len_mult */
         double len_ratio;  /* Rapport de taille entre le tampon original et le nouveau */
         void (*filters[10])(struct SDL_AudioCVT *cvt, Uint16 format);
         int filter_index;               /* Filtre de conversion en cours */
 } SDL_AudioCVT;
 {% endhighlight %}

Rassurez-vous, une bonne partie des champs de cette structure est remplie pour vous par SDL. Connaissant les caractéristiques des formats d'origine et de destination, il faut faire appel à la fonction `SDL_BuildAudioCVT()` afin de préparer le bloc de conversion :

{% highlight c %}
int SDL_BuildAudioCVT(SDL_AudioCVT *cvt, Uint16 src_format, Uint8 src_channels, int src_rate, Uint16 dst_format, Uint8 dst_channels, int dst_rate);
{% endhighlight %}

Son utilisation est simple : on fournit l'adresse d'un objet `SDL_AudioCVT` non initialisé, ainsi que les caractéristiques des formats audio : format (voir les constantes définies plus haut), nombre de canaux (1 ou 2 pour mono ou stéréo) et fréquence d'échantillonnage. La fonction renvoie une valeur nulle, si tout s'est bien déroulé.

Le bloc de conversion ainsi obtenu n'est cependant pas complet. Avant de pouvoir effectuer la conversion par le biais de la fonction `SDL_ConvertAudio()`, il faut remplir les champs `buf` et `len` de la structure. Attention, la taille effective du tampon est souvent supérieure à la taille des données d'origine, SDL effectuant la conversion in situ. La marche à suivre est donc d'allouer un tampon (par exemple, à l'aide de malloc) d'une taille de `len` (la taille des données d'origine) multipliée par `len_mult` (fourni par `SDL_BuildAudioCVT`), puis de charger les données au début du tampon ainsi alloué. Il est à noter que la valeur de `len` doit bien être la taille des données d'origine, sans tenir compte du facteur `len_mult` !

La conversion des données proprement dite est ensuite fort simple : il suffit de passer l'objet `SDL_AudioCVT` ainsi construit à la fonction `SDL_ConvertAudio()`.

### Mixer des échantillons

SDL permet également de mélanger les données de deux échantillons de même format (plus précisément les échantillons doivent avoir le même format audio utilisé pour la restitution). Pour mélanger des échantillons de formats différents, il faudra donc tout d'abord convertir les données vers le format de sortie, par l'intermédiaire du procédé décrit ci-dessus. Il faut cependant garder à l'esprit que ces fonctions de conversion et de mixage sont principalement destinées à être appelées depuis la fonction callback utilisée par SDL, lorsque les tampons audio de sortie doivent être remplis...

{% highlight c %}
void SDL_MixAudio(Uint8 *dst, Uint8 *src, Uint32 len, int volume); 
{% endhighlight %}

L'utilisation de cette fonction est assez évidente : on fournit les adresses des tampons source et destination (qui doivent avoir la même taille, car de même format). Le paramètre `volume` permet d'ajuster le volume de l'échantillon résultant et peut avoir des valeurs comprises entre 0 et 128 (il est recommandé d'utiliser la constante **SDL_MIX_MAXVOLUME** pour le volume maximum).

Enfin, n'oubliez pas d'appeler la fonction `SDL_CloseAudio()` lorsque vous n'avez plus besoin de jouer des sons. Cette fonction ferme les descripteurs de fichiers et termine la thread dédiée à la gestion du son.

### Exemple : jouer un fichier WAV

Il n'est pas évident de comprendre au premier abord les interactions des fonctions que nous venons d'examiner. La simplicité d'utilisation est pourtant appréciable, surtout comparée à des interfaces comme *DirectSound*... Le programme de l'encadré 1 charge deux fichiers WAV arbitraires en mémoire et les mélange en temps réel (les sons peuvent être de formats différents!).

### Librairie annexe : SDL Mixer

Afin de faciliter davantage la gestion du son, la bibliothèque annexe '*SDL Mixer*', fonctionnant directement au dessus de SDL Audio, fournit une interface plus avancée permettant entres autres :

* Un canal de musique dédié, supportant les musiques au format WAV, MP3 (via la librairie SMPEG), MOD et autres formats de modules soundtrack (via *Mikmod*) et MIDI (via *Timidity*),
* Un nombre arbitraire de canaux audio séparés, chacun pouvant jouer un son au format arbitraire. Chaque canal peut être stoppé, mis en pause, remis à zéro, peut faire l'effet d'un fondu, ...
* Volumes séparés pour chaque canal,
* Possibilité d'affecter une fonction callback à un canal.

Bref cette bibliothèque est le compagnon idéal du programmeur de jeu. Elle a été utilisée avec succès pour le portage de plusieurs jeux Loki (notamment, *Civilization : Call to Power*, *Railroad Tycoon II* et *Heroes of Might & Magic III*). 
Son API est très intuitive et vous pouvez obtenir plus d'informations en la téléchargeant sur le site Web de SDL: [libsdl.org](https://www.libsdl.org/)

### Conclusion

Nous arrivons pratiquement à la fin de cette initiation à la programmation SDL. D'ores et déjà, nous avons couvert tous les sous-systèmes les plus importants. Cependant, SDL fournit quelques autres mécanismes qui, s'ils ne sont pas des atouts majeurs, n'en facilitent pas moins grandement la tâche du programmeur. Parmi ceux-ci, nous trouvons la gestion des threads, des verrous d'exclusion mutuelle, des fonctions d'accès au lecteur de CDROM, les timers, la gestion des différentes architectures, etc. Tout ceci sera abordé dans notre prochain et dernier article, ainsi que les nouveautés présentes dans la toute nouvelle version 1.1 de SDL, bientôt appelée à devenir 1.2 ...

*Stéphane Peter*

----

### Encadré 1

{% highlight c %}
#include <SDL_audio.h>
#include <stdlib.h>
#include <stdio.h>

SDL_AudioSpec sortie, obtenu, wav1, wav2;
SDL_AudioCVT  wav1_cvt, wav2_cvt;

Uint8 *wav1_buf, *wav2_buf;
Uint32 wav1_len, wav2_len;

Uint8 *cvt1_buf, *cvt2_buf;

int index1 = 0, index2 = 0;

/* Fonction appelée par SDL pour remplir un buffer audio */
void fill_audio(void *udata, Uint8 *stream, int len)
{
  memset(stream,0,len);
  /* Mélange des échantillons, volume 75% */
  if(index1 < wav1_cvt.len_cvt) {
	if((index1+len) > wav1_cvt.len_cvt)
	  len = wav1_cvt.len_cvt - index1;
	SDL_MixAudio(stream, cvt1_buf + index1, len, 0.75 * SDL_MIX_MAXVOLUME);
	index1 += len;
  }
  if(index2 < wav2_cvt.len_cvt) {
	if((index2+len) > wav2_cvt.len_cvt)
	  len = wav2_cvt.len_cvt - index2;
	SDL_MixAudio(stream, cvt2_buf + index2, len, 0.75 * SDL_MIX_MAXVOLUME);
	index2 += len;
  }
}

#define AUDIO_BUFFER_SIZE 1024

int main(int argc, char **argv)
{
  if(argc < 3) {
	printf("Utilisation: %s son1.wav son2.wav\n", argv[0]);
	return 1;
  }

  if(!SDL_LoadWAV(argv[1], &wav1, &wav1_buf, &wav1_len) ||
	 !SDL_LoadWAV(argv[2], &wav2, &wav2_buf, &wav2_len)) {
	printf("Erreur lors du chargement d'un des fichiers WAV\n");
	return 1;
  }

  /* On demande un format de sortie 16 bits stéréo à 22kHz */
  sortie.freq = 22050;
  sortie.format = AUDIO_S16;
  sortie.channels = 2;
  sortie.samples = AUDIO_BUFFER_SIZE;
  sortie.callback = fill_audio;
  sortie.userdata = NULL;

  if ( SDL_OpenAudio(&sortie, &obtenu) < 0 ) {
	fprintf(stderr, "Erreur SDL Audio: %s\n", SDL_GetError());
	return 2;
  }

  /* Préparation des blocs de conversion audio */

  SDL_BuildAudioCVT(&wav1_cvt, wav1.format, wav1.channels, wav1.freq,
					obtenu.format, obtenu.channels, obtenu.freq);
  SDL_BuildAudioCVT(&wav2_cvt, wav2.format, wav2.channels, wav2.freq,
					obtenu.format, obtenu.channels, obtenu.freq);

  /* Conversion des échantillons vers le format de sortie */

  cvt1_buf = (Uint8*)malloc(wav1_len * wav1_cvt.len_mult);
  cvt2_buf = (Uint8*)malloc(wav2_len * wav2_cvt.len_mult);
  wav1_cvt.len = wav1_len;
  wav1_cvt.buf = cvt1_buf;
  wav2_cvt.len = wav2_len;
  wav2_cvt.buf = cvt2_buf;
  memcpy(cvt1_buf, wav1_buf, wav1_len);
  memcpy(cvt2_buf, wav2_buf, wav2_len);

  SDL_ConvertAudio(&wav1_cvt);
  SDL_ConvertAudio(&wav2_cvt);

  /* Démarrage de la restitution (en tâche de fond) */
  SDL_PauseAudio(0);
  /* Attend la fin de la restitution */
  while((index1 < wav1_cvt.len_cvt) || (index2 < wav2_cvt.len_cvt)) {
	SDL_Delay(100);
  }
  free(cvt1_buf); free(cvt2_buf);
  SDL_FreeWAV(wav1_buf);
  SDL_FreeWAV(wav2_buf);
  SDL_CloseAudio();
  return 0;
}
{% endhighlight %}
