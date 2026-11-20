# ReceptionCRM

Laravel 13 projektas, automatiškai diegiamas į CloudPanel serverį per GitHub Actions.

## Vietinis paleidimas

```bash
composer install
cp .env.example .env
php artisan key:generate
php artisan migrate
npm install && npm run build
php artisan serve
```

## Diegimas į CloudPanel

Kiekvienas `push` į `master` paleidžia `.github/workflows/deploy.yml`, kuris per SSH
prisijungia prie serverio ir atnaujina kodą (`git reset --hard origin/master`,
`composer install`, `npm run build`, migracijos, cache).

### 1. Svetainė CloudPanel'e

1. CloudPanel → **Add Site** → **Create a PHP Site** → Application: **Laravel**, PHP 8.3+.
2. Įsiminkite site user (pvz. `reception`) ir kelią, pvz.
   `/home/reception/htdocs/reception.example.lt`.
3. Document root turi būti `.../public` (Laravel šablonas tai nustato pats).
4. Susikurkite duomenų bazę (**Databases** skiltis), jei naudosite MySQL.

### 2. Pirmas kodo įkėlimas į serverį

Prisijunkite per SSH kaip site user ir sukurkite deploy raktą GitHub'ui:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -N ""
cat ~/.ssh/github_deploy.pub
```

Viešą raktą įdėkite į GitHub → repo **Settings → Deploy keys** (tik skaitymo teisės).
Tada `~/.ssh/config`:

```
Host github.com
  IdentityFile ~/.ssh/github_deploy
```

Klonuokite projektą į svetainės aplanką (pirma ištrinkite CloudPanel sukurtus failus):

```bash
cd ~/htdocs/reception.example.lt
rm -rf ./* ./.??*
git clone git@github.com:richa22/ReceptionCRM.git .
composer install --no-dev --optimize-autoloader
cp .env.example .env
php artisan key:generate
```

`.env` faile nustatykite `APP_ENV=production`, `APP_DEBUG=false`, `APP_URL` ir DB duomenis, tada:

```bash
php artisan migrate --force
npm ci && npm run build
```

### 3. GitHub Actions slaptažodžiai

Sukurkite atskirą raktą, kuriuo GitHub jungsis prie serverio (savo kompiuteryje):

```bash
ssh-keygen -t ed25519 -f cloudpanel_deploy -N ""
```

`cloudpanel_deploy.pub` turinį pridėkite į serverio `~/.ssh/authorized_keys`
(arba CloudPanel → Site → **SSH/FTP** → SSH Keys).

GitHub → repo **Settings → Secrets and variables → Actions** sukurkite:

| Secret        | Reikšmė                                         |
|---------------|-------------------------------------------------|
| `SSH_HOST`    | serverio IP arba domenas                        |
| `SSH_PORT`    | SSH portas (nebūtina, numatytasis 22)            |
| `SSH_USER`    | site user, pvz. `reception`                     |
| `SSH_KEY`     | privataus rakto `cloudpanel_deploy` turinys      |
| `DEPLOY_PATH` | pvz. `/home/reception/htdocs/reception.example.lt` |

Po to kiekvienas `push` į `master` (arba rankinis paleidimas **Actions → Deploy to CloudPanel → Run workflow**) atnaujins svetainę.
