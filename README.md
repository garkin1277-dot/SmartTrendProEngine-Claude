# OKX Agent Skills

Набор навыков для AI-агентов, который помогает работать с биржей OKX через CLI `okx`. Навыки содержат инструкции для получения рыночных данных, управления аккаунтом и ордерами, а также настройки торговых ботов — без необходимости самостоятельно подключать API в каждом агенте.

> Это README описывает проект [`okx/agent-skills`](https://github.com/okx/agent-skills). Перед использованием проверьте актуальные инструкции и требования в исходном репозитории.

## Возможности

| Навык | Что делает | Требуется авторизация |
|---|---|:---:|
| [okx-cex-market](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-market/SKILL.md) | Рыночные данные: цены, стаканы, свечи, ставки финансирования и другие показатели | Нет |
| [okx-cex-trade](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-trade/SKILL.md) | Управление ордерами и торговыми операциями | Да |
| [okx-cex-portfolio](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-portfolio/SKILL.md) | Балансы, позиции, P&L, комиссии и переводы средств | Да |
| [okx-cex-bot](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-bot/SKILL.md) | Настройка и управление grid- и DCA-ботами | Да |
| [okx-cex-earn](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-earn/SKILL.md) | Доступ к продуктам OKX Earn | Да |
| [okx-cex-smartmoney](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-smartmoney/SKILL.md) | Аналитика и отслеживание активности трейдеров | Да |
| [okx-sentiment-tracker](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-sentiment-tracker/SKILL.md) | Новости, настроения рынка и популярные активы | Да |
| [okx-cex-skill-mp](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills/okx-cex-skill-mp/SKILL.md) | Поиск и управление наборами навыков в OKX Skills Marketplace | Да |

## Требования

- Установленный CLI [`@okx_ai/okx-trade-cli`](https://www.npmjs.com/package/@okx_ai/okx-trade-cli).
- Для навыков, которым нужна авторизация, — настроенный доступ к аккаунту OKX.

Установите CLI:

```bash
npm install -g @okx_ai/okx-trade-cli
```

Для настройки доступа можно использовать OAuth через навык `okx-cex-auth` или настроить CLI вручную:

```bash
okx config init
```

Следуйте актуальным инструкциям в [исходном README](https://github.com/okx/agent-skills/blob/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/README.md#requirements).

## Использование

Навыки предназначены для AI-агентов, совместимых с форматом Skills. Каждый навык — отдельный файл `SKILL.md` с описанием задач, для которых его следует активировать, и инструкциями по использованию CLI.

Список навыков и их инструкции находятся в каталоге [`skills/`](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills).

## Сведения о репозитории

- **Исходный репозиторий:** [okx/agent-skills](https://github.com/okx/agent-skills)
- **Repo ID:** `1174065443`
- **Основной язык:** Shell (100%)
- **Лицензия:** MIT — подробности в файле [LICENSE](https://github.com/okx/agent-skills/blob/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/LICENSE)

## Важно

Навыки могут выполнять операции, связанные с торговлей и управлением аккаунтом. Перед запуском проверяйте команды и параметры, а секреты API храните безопасно. Информация в этом README не является финансовой рекомендацией.

## Ссылки

- [Исходный репозиторий](https://github.com/okx/agent-skills)
- [Исходный README](https://github.com/okx/agent-skills/blob/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/README.md)
- [Каталог навыков](https://github.com/okx/agent-skills/tree/3cff92b6a5ced87422fbfe0ce4d7237c026589f9/skills)