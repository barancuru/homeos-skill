# HomeOS skill'i

[HomeOS](https://home.curu.tr) (ev yönetimi: envanter, garanti, bakım, belgeler, ortak gider, tekrarlayan ödemeler, kişisel bütçe) için Claude skill'i. Claude, HomeOS'u kendi yetkinle okur, kayıt ekler ve senin onayınla arşivler ya da siler.

## Kurulum
Ayrıntılı adımlar: [`homeos/KURULUM.md`](homeos/KURULUM.md). Kısaca:

1. **Skill'i kur** (Claude Code):
   ```bash
   git clone git@github.com:barancuru/homeos-skill.git
   mkdir -p ~/.claude/skills && cp -r homeos-skill/homeos ~/.claude/skills/
   ```
   Güncellemek için: `git -C homeos-skill pull && cp -r homeos-skill/homeos ~/.claude/skills/`
2. **HomeOS'u bağla** (anahtar gerekmez):
   - claude.ai / Claude Desktop: Ayarlar → Bağlayıcılar → Özel bağlayıcı ekle → `https://home.curu.tr/mcp`
   - Claude Code: `claude mcp add --transport http homeos https://home.curu.tr/mcp`, sonra `/mcp` → homeos → kimlik doğrula. claude.ai'de bağlayıcı zaten varsa bu adıma gerek yok.
3. Sor: *"HomeOS'ta önümüzdeki 3 ayda garantisi bitecek ya da bakımı gelecek neler var?"*

## İçerik
| Dosya | Ne |
|---|---|
| `homeos/SKILL.md` | Claude'un ne zaman ve nasıl kullanacağı (okuma, ekleme, arşivleme ve silme kuralları) |
| `homeos/references/api.md` | REST uçları, MCP araçları, alanlar, limitler, hatalar |
| `homeos/scripts/homeos-api` | Bağlayıcı kullanılamazsa `hos_` anahtarıyla çalışan bash + curl istemcisi |
| `homeos/KURULUM.md` | İnsan için kurulum rehberi |

Sürüm uyumu `homeos/references/api.md` başındaki satırda yazar. Canlı şema her zaman `https://home.curu.tr/api/openapi.json` ve MCP `tools/list`'tir.
