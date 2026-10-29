# Official ESPHome distribution: compiler, ESP-IDF, esptool and OTA CLI included.
FROM ghcr.io/esphome/esphome:2026.9.0
ARG BUILD_VERSION
ARG BUILD_ARCH
# The Supervisor passes BUILD_ARCH (aarch64, amd64). The image GitHub builds for Docker (docker.yml) builds both
# platforms at once, so there Docker's own TARGETARCH (arm64, amd64) fills the label.
ARG TARGETARCH
LABEL io.hass.version="${BUILD_VERSION}" io.hass.type="app" io.hass.arch="${BUILD_ARCH:-$TARGETARCH}"
# A camera live on a P4 screen (camera_source.py): aiortc receives Home Assistant's WebRTC of a camera, and brings PyAV
# (FFmpeg) to decode its H.264 and make its JPEGs. Every Home Assistant builds this image itself, so the WebRTC part
# is pinned to the set pip resolves on this ESPHome for amd64 and aarch64 (wheels on Python 3.12 and 3.13), and ESPHome's
# own cryptography is a constraint: a later ESPHome that no longer fits fails this build instead of losing its pin.
RUN (pip freeze | grep -iE '^(cryptography|pillow)==' || true) > /tmp/esphome-keeps.txt \
 && pip install --no-cache-dir -c /tmp/esphome-keeps.txt 'aiohttp>=3.12,<4' PyYAML \
      aiortc==1.15.0 av==17.1.0 aioice==0.10.2 pylibsrtp==1.0.0 pyee==14.0.0 pyOpenSSL==26.4.0 google-crc32c==1.9.0 \
      pillow-heif==1.8.0 \
 && rm /tmp/esphome-keeps.txt
COPY app /app
# Every text of the app, the editor and the screens, one file per language (app 0.2.90).
COPY translations /app/translations
COPY CHANGELOG.md /app/CHANGELOG.md

# TEST-BRANCH ONLY: expose the Makerfabs 4.3 V3.1 in New screen before the generated boards.json is committed.
# Remove this block before the upstream PR and regenerate boards.json normally.
RUN python3 - <<'PY'
import copy
import json

path = '/app/boards.json'
with open(path, encoding='utf-8') as f:
    boards = json.load(f)

base = copy.deepcopy(boards['waveshare43'])
base['board'] = 'makerfabs43'
# Test hardware is the R59-fitted/R29-removed V3.1 variant, so GPIO44 drives the backlight with PWM.
base['dimmable'] = True
base['can_standby'] = True
catalog = copy.deepcopy(base.get('catalog', {}))
catalog.update({
    'name': 'Makerfabs',
    'model': 'E32S3RGB43 V3.1',
    'status': 'experimental',
    'choices': {},
    'settings': [],
})
catalog['order'] = max(
    (shape.get('catalog', {}).get('order', -1) for shape in boards.values()),
    default=-1,
) + 1
base['catalog'] = catalog

for key in ('makerfabs43', 'packages/makerfabs43.yaml', 'checkout/makerfabs43.yaml'):
    boards[key] = copy.deepcopy(base)

with open(path, 'w', encoding='utf-8') as f:
    json.dump(boards, f, indent=1, ensure_ascii=False)
    f.write('\n')
PY

# TEST-BRANCH ONLY: a fork has no built-in Tessera channel, so point new screen builds at this fork and branch.
# Remove these replacements before the upstream PR.
RUN sed -i \
  -e "s|REPO = 'https://github.com/MaxGramser/homeassistant_espscreen'|REPO = 'https://github.com/lubu2804/homeassistant_espscreen'|" \
  -e "s|REF = 'main'|REF = 'feat/makerfabs43-v31'|" \
  /app/core.py

WORKDIR /app
# The ESPHome image checks its dashboard on port 6052, which this app never starts, so the container read
# "unhealthy" while everything worked (issue #23). This one asks our own server, on the port the page is on;
# under the Supervisor every other address gets a 403, so it is /health and not /.
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD curl --fail --silent --output /dev/null http://127.0.0.1:8099/health || exit 1
# The image GitHub builds from dev for Docker is the dev app (docker.yml passes CHANNEL=dev), which an app without a
# Supervisor reads (core.start_channel). Empty everywhere else: under the Supervisor the app's slug says it. Last, so
# dev and main share every layer above.
ARG CHANNEL=""
ENV SCREEN_CHANNEL="${CHANNEL}"
ENTRYPOINT []
CMD ["python3", "/app/server.py"]
