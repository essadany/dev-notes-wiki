# Dev notes wiki

Base de connaissances technique personnelle — notes issues de cours, de playlists et de concepts que je veux pouvoir revoir plus tard, sans avoir à revisionner la source.

## Sujets

- **[Sujets classiques d'une app web pro](global-subjects/global-subjects.md)** — 40 fiches (frontend, backend, architecture) avec résumé, pièges et exemples techniques. Point d'entrée pour retrouver rapidement un concept.
- **[Backend from first principles](backend-from-first-principles/index.md)** — notes issues de [cette playlist](https://www.youtube.com/playlist?list=PLui3EUkuMTPgZcV0QhQrOcwMPcBCcd_Q1), qui explore les systèmes backend à partir des concepts les plus fondamentaux.
- **[Langages de programmation](langages/index.md)** — événements, boucle, concurrence et parallélisme à travers JS/TS, Python, C, C++, Java et PHP.
- **[Déploiement web](deploiement-web/index.md)** — EC2, DNS, ports, reverse proxy Nginx et SSL avec Certbot.

## Organisation

Chaque sujet possède son propre dossier sous `docs/` :

- Les contenus issus d'une playlist sont découpés vidéo par vidéo, dans l'ordre.
- Les « Sujets classiques » forment un hub : une page par sujet, accessible depuis le sommaire.
- La navigation latérale (onglets et sections) est définie dans `mkdocs.yml`.
