# OpenCode Skills

Vijf skills voor [OpenCode](https://opencode.ai): korte, geteste werkwijzen die het model automatisch laadt zodra een taak erbij past. Elke skill is één `SKILL.md` met een naam, een beschrijving en de stappen die in de praktijk werken.

| Skill | Waarvoor |
| --- | --- |
| [`android-apk-no-gradle`](android-apk-no-gradle/SKILL.md) | een Android-APK bouwen zonder Gradle: aapt2, javac, d8, apksigner |
| [`android-phone-modding`](android-phone-modding/SKILL.md) | rooten, bootloader unlocken, flashen, firmware dumpen, mtkclient |
| [`local-llm-serving`](local-llm-serving/SKILL.md) | een lokaal taalmodel draaien: llama.cpp, vllm, ollama, geen GPU |
| [`quickshell-omarchy-plugin`](quickshell-omarchy-plugin/SKILL.md) | een Omarchy/Quickshell bar-plugin maken, installeren en debuggen |
| [`zero-dep-python-tool`](zero-dep-python-tool/SKILL.md) | een Python-CLI of -service met alleen de standaardbibliotheek |

## Installeren

Kopieer de mappen naar je OpenCode-config:

```bash
git clone https://github.com/michielturmel-max/opencode-skills.git
mkdir -p ~/.config/opencode/skills
cp -r opencode-skills/* ~/.config/opencode/skills/
```

Of symlink ze, zodat een `git pull` meteen doorwerkt:

```bash
git clone https://github.com/michielturmel-max/opencode-skills.git
mkdir -p ~/.config/opencode/skills
for d in opencode-skills/*/; do
  ln -sfn "$(cd "$d" && pwd)" ~/.config/opencode/skills/"$(basename "$d")"
done
```

OpenCode leest de skills bij het starten. Herstart een lopende sessie als je er
een hebt toegevoegd. Je kunt een skill ook expliciet noemen, bijvoorbeeld:
"gebruik de skill `zero-dep-python-tool`".

## Zelf een skill maken

Een skill is een map met daarin `SKILL.md`:

```markdown
---
name: mijn-skill
description: Wanneer deze skill gebruikt moet worden, in één of twee zinnen.
---

De stappen, valkuilen en commando's.
```

De `description` is wat OpenCode ziet voordat het de skill laadt; houd hem
concreet en noem de trefwoorden waarop je wilt matchen.

## Licentie

MIT, zie [LICENSE](LICENSE).
