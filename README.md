# BigBlueButton Attendance for Moodle

**Version:** `0.2.0-alpha` · **Status:** Alpha — test in a staging environment before production use.

This project contains two Moodle plugins. Install both for the complete course activity workflow:

- **`local_bbattendance.zip`** receives BigBlueButton webhooks, stores attendance data in Moodle's database, displays a course report, and exports CSV.
- **`mod_bbattendance.zip`** adds the **BBB attendance report** activity to a Moodle course and opens the local plugin's report.

## Requirements

### Moodle server

- **Moodle 5.2.x** is the target for this alpha release; other versions have not been independently verified.
- PHP **8.3 or 8.4**, 64-bit, as supported by Moodle 5.2.
- A supported Moodle database, already configured and working.
- The official Moodle **BigBlueButtonBN** activity plugin installed and configured.
- Moodle administrator access to install plugins.
- HTTPS strongly recommended. The BBB server must be able to reach the Moodle webhook endpoint over HTTPS.

See the [official Moodle 5.2 requirements](https://moodledev.io/general/releases/5.2).

### BigBlueButton server

- A working BigBlueButton server integrated with Moodle.
- The official **`bbb-webhooks`** service installed and running.
- Webhooks configured to send processed events (not raw payloads) for `user-joined`, `user-left`, and `meeting-ended`.
- Outbound HTTPS connectivity to `https://YOUR-MOODLE-DOMAIN/local/bbattendance/webhook.php`.
- Permission to configure and restart `bbb-webhooks`.

The plugin does not need direct access to the Moodle database from the BBB server. Do not expose the database to BBB for this plugin.

References: [BigBlueButton Webhooks](https://docs.bigbluebutton.org/development/webhooks/) · [official bbb-webhooks repository](https://github.com/bigbluebutton/bbb-webhooks).

## Installation

### 1. Back up Moodle

Back up the Moodle code and database. Test the installation on a staging site first.

### 2. Install `local_bbattendance`

1. In Moodle, open **Site administration → Plugins → Install plugins**.
2. Upload `local_bbattendance.zip`.
3. Open **Site administration → Notifications** to complete installation and create the plugin's database tables.
4. Open **Site administration → Plugins → Local plugins → BigBlueButton Attendance settings**.
5. Configure:
   - **BBB hostname:** hostname only, e.g. `bbb.example.org`; do not include `https://` or a path.
   - **Webhook token:** the same secret sent by `bbb-webhooks` in the HTTP `Authorization: Bearer ...` header.

Keep the secret private. Never commit it to GitHub or put it in a public URL.

### 3. Configure `bbb-webhooks` on the BBB server

Set the BBB domain and shared secret in the existing `bbb-webhooks` configuration, and enable Bearer authentication:

```yaml
bbb:
  serverDomain: bbb.example.org
  sharedSecret: "REPLACE_WITH_A_LONG_RANDOM_SECRET"
  auth2_0: true
```

The `sharedSecret` must match the **Webhook token** saved in Moodle.

In the existing configuration for `../out/webhooks/index.js`, add the Moodle callback as a permanent webhook. Merge this with the current configuration; do not replace other required settings:

```yaml
modules:
  ../out/webhooks/index.js:
    enabled: true
    type: out
    config:
      permanentURLs:
        - url: "https://YOUR-MOODLE-DOMAIN/local/bbattendance/webhook.php"
          getRaw: false
```

Important:

- Preserve the existing `modules` configuration, including Redis and other required settings.
- `getRaw: false` is required because this plugin expects processed event payloads.
- The request body must contain form fields named `domain` and `event`. The JSON in `event` must include `data.id`, `data.attributes`, and `data.event.ts`.
- The hostname saved in Moodle must match the webhook's `domain` field.
- `auth2_0: true` enables Bearer authentication. Use the same secret on both servers.

Restart `bbb-webhooks` after changing its configuration. Use the service-management command appropriate to your installation and consult the official documentation if needed.

### 4. Install `mod_bbattendance` (recommended)

After the local plugin is installed:

1. Upload `mod_bbattendance.zip` through **Site administration → Plugins → Install plugins**.
2. Complete installation through **Site administration → Notifications**.
3. Open a Moodle course, turn editing on, and add **BBB attendance report**.
4. Test access using the roles that should be able to view reports.

The local plugin can display reports by itself; the activity plugin provides a course-page shortcut.

## Test the setup

1. Open a Moodle course with a BigBlueButtonBN activity.
2. Start a test meeting from Moodle.
3. Join as a non-guest Moodle user, leave, and optionally rejoin.
4. End the meeting.
5. Check the attendance report and CSV export.
6. If no data appears, check Moodle and `bbb-webhooks` logs.

The external meeting ID must match the format expected by this release:

`<seed>-<courseid>-<instanceid>[groupid]`

The collector uses the numeric BBB `external-user-id` as the Moodle user ID. Guests and users with non-numeric external IDs are not linked to Moodle accounts.

## Known limitations

- Alpha release; compatibility across Moodle and BBB versions has not been independently verified.
- Events from before webhook activation are not imported retrospectively.
- Meeting ID formats and webhook payloads may vary between integrations and versions; test with your environment.
- Attendance records are personal data. Configure access and retention according to your institution's policies.
- This is not an official or endorsed Moodle or BigBlueButton plugin.

## Troubleshooting

**Empty report**
- Confirm both plugins installed successfully and the course uses the official Moodle BigBlueButtonBN activity.
- Verify `bbb-webhooks` is running and the callback URL is correct.
- Confirm `getRaw: false`, the matching BBB hostname, and the same Bearer token on both servers.
- Verify the meeting ID format and that `external-user-id` is a numeric Moodle user ID.

**`401 Unauthorized`**
- Check `auth2_0: true` and that the token matches on both sides.
- Ensure the web server forwards the `Authorization` header to PHP.

**`403 Unexpected BBB domain`**
- Set the BBB hostname in Moodle to match the webhook's `domain` field, without scheme or path.

**Webhook cannot connect**
- Check DNS, HTTPS certificate validity, reverse-proxy configuration, firewall rules, and outbound connectivity from BBB to Moodle.

## Security

- Use HTTPS for webhook delivery.
- Never publish the webhook token or BBB shared secret.
- Do not commit production configuration files or real attendance records.
- Restrict report access to authorized users and review data-retention requirements.

## License

Add a `LICENSE` file before public release. Choose a license you are authorized to apply and check the licenses of any third-party code included in the repository.
