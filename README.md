# Spider Lily Archive

## About

This project is an open-source discography and credit archive for Ado's original songs and covers.

If a song is a cover, it stores links to both the original and Ado's version alongside Ado's version credits.

## How it works

The project has two layers: a source of truth you edit by hand, and a database that serves the site.

- Adding a song is done through `tools/add-song.php`, a local-only form. Submitting it appends one entry to `data/songs.json`.
- `php tools/sync.php` reads that JSON and upserts it into MariaDB, matching songs by `slug`. It is safe to run as often as you like: existing rows are updated, new ones inserted.
- The public pages read only from the database. `public/index.php` lists the catalog, `public/song.php` renders a single song with its credits and links, and `public/api/search.php` backs the live search box.


### Data model

`songs` holds what every song has: slug, the romaji/Japanese/English titles, `is_cover`, album and notes.

Each song has at most one row in each of these three tables, linked back to it by `song_id` in a one-to-one relationship. A row cannot exist without its song, so deleting a song deletes its rows as well.

| Table | Holds |
| :--- | :--- |
| `song_credits` | composer, lyricist, arranger, producer, performers and engineers |
| `song_ado_data` | Ado's release date, YouTube link, Niconico link |
| `song_original_data` | for covers only: the original artist, its release date and links |

Keeping them separate leaves `songs` small. `songs.is_cover` is what the site checks to decide whether a song is a cover, and a `song_original_data` row is only written for songs where `is_cover = 1`. Because syncing never deletes rows, a `song_original_data` row can outlive a song's cover status. `is_cover` is always the source of truth.

## Features

- Browse Ado's discography.
- View individual song pages with detailed credits.
- Distinguish original songs from covers.
- Access links to Ado's releases and original versions of covers.
- Search the catalog using live search.

## Architecture

Only `public/` is exposed as the Nginx document root, keeping configuration, database logic and CLI scripts outside the web root. Clean song URLs `/songs/<slug>` are rewritten internally by Nginx to `/song.php?slug=<slug>`.

```text
├── config/     # config.php, copied from the example config but filled with real data.
├── data/       # songs.json, the dataset that holds songs information and can be edited by hand.
├── database/   # schema.sql, the database dump ready to import.
├── public/     # web root, files in there get served by Nginx.
├── src/        # db.php, shared PDO connection.
└── tools/      # add-song.php is a local form that writes into JSON, and sync.php imports that JSON into the database.
```

## Technologies

- Backend - PHP 8.x
- Database - MariaDB
- Frontend - HTML5, CSS3, Vanilla JavaScript
- Web Server - Nginx + PHP-FPM

## Data Progress

| Category | Release | Status | Count |
| :--- | :--- | :--- | :--- |
| Album | Kyōgen | Completed | 14/14 |
| Album | Uta no Uta | Completed | 8/8 |
| Album | Zanmu | In Progress | 0/16 |
| Album | Ado's Utattemita Album | Planned | 0/10 |
| Album | Ado's Utattemita Album Vol. 2 | Not released yet | ?/? |
| Album | Ado's Best Adobum Disc 1 | In Progress | 10/20 |
| Album | Ado's Best Adobum Disc 2 | In Progress | 7/20 |
| Singles | Standalone Single Releases | Planned | 0/23 |
| Covers | Youtube / Niconico Covers | Planned | 1/112? |
| Features | Songs where Ado is featured | Planned | 0/28 |
| Extras | Special songs | Planned | 0/13 |

## Setup

This guide assumes a Linux environment and walks through running the site **locally** with Nginx + PHP-FPM. It does not cover deploying to a server.

1. Clone the repo:

```bash
git clone https://github.com/sseraflm/spider-lily-archive.git
cd spider-lily-archive
```

2. Create the MariaDB database and user, and import the schema (replace `your_password` with your own). MariaDB must already be installed and running:

```bash
sudo mariadb -e "CREATE DATABASE spider_lily_archive CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
sudo mariadb -e "CREATE USER 'spiderlily'@'localhost' IDENTIFIED BY 'your_password'; GRANT ALL PRIVILEGES ON spider_lily_archive.* TO 'spiderlily'@'localhost';"
sudo mariadb spider_lily_archive < database/schema.sql
```

3. Configure database credentials:

```bash
cp config/config.example.php config/config.php
```

Edit `config/config.php` with your database host, port, name, username, and password. Use `localhost` as the host so it matches the `'spiderlily'@'localhost'` grant above. Never commit this file — it is already listed in `.gitignore`.

4. Install PHP, PHP-FPM and the PDO MySQL driver:

```bash
# Debian / Ubuntu
sudo apt install php php-fpm php-mysql

# Arch
sudo pacman -S php php-fpm
```

On Arch the driver ships with the `php` package. Make sure `extension=pdo_mysql` is enabled in `/etc/php/php.ini`. Without the driver, `tools/sync.php` fails with "could not find driver".

5. Sync song data into the database:

```bash
php tools/sync.php
```

6. Configure Nginx:

Point your server block `root` to `public/` and add the rewrite rule for clean song URLs. Adjust the `fastcgi_pass` socket and `include` path for your distro:

```nginx
root /path/to/spider-lily-archive/public;
index index.php;

rewrite ^/songs/([a-zA-Z0-9_-]+)$ /song.php?slug=$1 last;

location / {
    try_files $uri $uri/ =404;
}

location ~ \.php$ {
    try_files $uri =404;
    fastcgi_pass unix:/run/php-fpm/php-fpm.sock;
    fastcgi_index index.php;
    include fastcgi.conf;
}
```

## Adding songs

`tools/add-song.php` is a local form that appends one entry to `data/songs.json`. It lives outside the Nginx web root, so it is not reachable through the site. Serve it separately with PHP's built-in server.

1. Make sure you are in the project root directory:

```bash
cd spider-lily-archive
```

2. Start the form:

```bash
php -S localhost:8000 -t tools
```

3. Open `http://localhost:8000/add-song.php`.

4. Fill in the fields. Slug, romaji title and release date are required; a cover also requires the original artist.

5. Hit Submit. The song will be appended to `data/songs.json`, and the page confirms with "Song added to data/songs.json."

6. Sync it with the database:

```bash
php tools/sync.php
```

If saving fails with "Could not open data/songs.json for writing", the user running `php -S` needs write permission on that file.

## License

This project is licensed under the GNU Affero General Public License v3.0 (AGPLv3). Any modified version hosted publicly over a network must also make its source code available under the same license.