<template>
  <div>
    <h2 class="mb-4">{{ $t('plugin_opencode.title') }}</h2>

    <v-skeleton-loader v-if="loading" :loading="true" type="card" />

    <div v-else style="margin-bottom: 80px">
      <!-- Status Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-robot</v-icon>
          <span>{{ $t('plugin_opencode.status') }}</span>
        </v-card-title>
        <v-card-text>
          <v-row>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_opencode.version') }} {{ currentVersion || $t('plugin_opencode.not_installed') }}
              </span>
            </v-col>
            <v-col cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_opencode.latest_version') }} {{ latestVersion || '-' }}
              </span>
            </v-col>
            <v-col v-if="webuiEnabled" cols="12" md="3">
              <span class="text-subtitle-1 font-weight-medium">
                {{ $t('plugin_opencode.status') }}: {{ running ? $t('plugin_opencode.running') : $t('plugin_opencode.not_running') }}
              </span>
            </v-col>
            <v-col cols="12" md="auto" class="d-flex ga-2 flex-nowrap flex-md-row">
              <v-btn size="small" variant="tonal" color="secondary" @click="checkForUpdates" :loading="checkingUpdates">
                {{ $t('plugin_opencode.check_updates') }}
              </v-btn>
              <v-btn
                v-if="!currentVersion && dataHomeSaved && settings.data_home"
                size="small" variant="tonal" color="primary"
                @click="doInstall" :loading="installing"
              >
                {{ $t('plugin_opencode.install') }}
              </v-btn>
              <v-btn
                v-else-if="currentVersion && updateAvailable"
                size="small" variant="tonal" color="primary"
                @click="doUpdate" :loading="updating"
              >
                {{ $t('plugin_opencode.update') }}
              </v-btn>
            </v-col>
          </v-row>
        </v-card-text>
      </v-card>

      <!-- Settings Card -->
      <v-card class="mb-4 pa-0">
        <v-card-title class="d-flex align-center">
          <v-icon class="mr-2">mdi-cog</v-icon>
          <span>{{ $t('plugin_opencode.settings') }}</span>
        </v-card-title>
        <v-card-text>
          <v-text-field
            v-model="settings.data_home"
            :label="$t('plugin_opencode.data_home')"
            :hint="!dataHomeSaved ? $t('plugin_opencode.data_home_hint', { path: recommendedPath }) : ''"
            persistent-hint
            class="mt-2"
          >
            <template #append-inner>
              <v-btn size="small" icon="mdi-folder" @click="openFsDialog" variant="text" />
            </template>
          </v-text-field>
          <div v-if="!dataHomeSaved" class="text-subtitle-2 text-warning mt-2">
            {{ $t('plugin_opencode.no_data_home_set') }}{{ recommendedPath ? ' ' + recommendedPath : '' }}
          </div>
          <template v-if="dataHomeSaved">
            <v-divider class="my-4" />
            <v-text-field
              v-model="settings.hostname"
              :label="$t('plugin_opencode.hostname')"
              hint="Default: 0.0.0.0"
              persistent-hint
              style="max-width: 200px"
            />
            <v-text-field
              v-model="settings.port"
              :label="$t('plugin_opencode.port')"
              :hint="webuiEnabled ? $t('plugin_opencode.port_active') : $t('plugin_opencode.port_disabled')"
              persistent-hint
              class="mt-4"
              type="number"
              style="max-width: 200px"
            />
            <v-switch
              v-if="webuiEnabled"
              v-model="settings.auto_start"
              :label="$t('plugin_opencode.auto_start')"
              inset color="green" hide-details
              class="mt-4"
            />
          </template>
          <v-btn color="primary" class="mt-4" @click="saveSettings" :loading="saving">
            <v-icon start>mdi-content-save</v-icon>
            {{ $t('plugin_opencode.save_settings') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Start/Stop Card -->
      <v-card v-if="currentVersion && dataHomeSaved" class="mb-4 pa-0">
        <v-card-text class="d-flex align-center ga-2">
          <v-btn color="primary" rounded :loading="starting" @click="startDaemon">
            <v-icon start>mdi-play</v-icon>
            {{ $t('plugin_opencode.start') }}
          </v-btn>
          <v-btn color="error" rounded variant="outlined" :loading="stopping" @click="stopDaemon">
            <v-icon start>mdi-stop</v-icon>
            {{ $t('plugin_opencode.stop') }}
          </v-btn>
          <v-btn
            v-if="running && webuiEnabled"
            color="secondary" rounded variant="tonal"
            :href="webuiUrl" target="_blank"
          >
            <v-icon start>mdi-open-in-new</v-icon>
            {{ $t('plugin_opencode.open_webui') }}
          </v-btn>
        </v-card-text>
      </v-card>

      <!-- Terminal Section -->
      <v-container v-if="currentVersion" fluid class="pt-2 pr-0 pl-0 pb-2">
        <div class="d-flex align-center ga-3 mb-4">
          <div style="width: 4px; height: 32px; border-radius: 2px; background: rgb(var(--v-theme-primary))"></div>
          <h2 class="font-weight-medium ma-0" style="font-weight: 600; line-height: 1.1">{{ $t('plugin_opencode.terminal') }}</h2>
          <v-spacer></v-spacer>
          <v-btn
            v-if="!terminalActive"
            text class="d-flex align-center" density="compact"
            @click="openTerminal"
          >
            <v-icon small class="mr-1">mdi-console</v-icon>
            {{ $t('plugin_opencode.open_terminal') }}
          </v-btn>
          <v-btn
            v-else
            text class="d-flex align-center" density="compact" color="error"
            @click="closeTerminal"
          >
            <v-icon small class="mr-1">mdi-close</v-icon>
            {{ $t('plugin_opencode.close_terminal') }}
          </v-btn>
        </div>
        <p v-if="!terminalActive" class="text-medium-emphasis ml-5">
          {{ $t('plugin_opencode.terminal_hint') }}
        </p>
        <div v-else>
          <div ref="terminalContainer" style="width: 100%; height: 840px; padding: 8px; background: #000000; border-radius: 8px"></div>
        </div>
      </v-container>
    </div>

    <!-- Overlay -->
    <v-overlay :model-value="overlay" class="align-center justify-center">
      <v-progress-circular color="onPrimary" size="64" indeterminate />
    </v-overlay>

    <!-- File System Dialog -->
    <v-dialog v-model="fsDialog" max-width="800">
      <v-card>
        <v-card-title class="d-flex align-center">
          <span>{{ $t('plugin_opencode.select_directory') }}</span>
          <v-spacer />
          <v-chip size="small" class="ml-2" variant="tonal">{{ fsCurrentPath || '/' }}</v-chip>
        </v-card-title>
        <v-card-subtitle class="pb-0">
          <div class="d-flex align-center ga-2">
            <v-btn size="small" variant="text" icon="mdi-home" @click="fsGoRoot" color="secondary" :disabled="fsLoading" />
            <v-btn size="small" variant="text" icon="mdi-arrow-up" @click="fsNavigateUp" color="secondary" :disabled="fsCurrentPath === '/mnt' || fsLoading" />
            <v-btn size="small" variant="text" icon="mdi-refresh" @click="fetchDirectory(fsCurrentPath)" color="secondary" :disabled="fsLoading" />
            <v-spacer />
            <v-progress-circular v-if="fsLoading" indeterminate size="20" color="secondary" />
          </div>
        </v-card-subtitle>
        <v-card-text class="pt-2" style="min-height: 300px; max-height: 60vh; overflow-y: auto">
          <v-table density="compact">
            <thead>
              <tr>
                <th>{{ $t('plugin_opencode.name') }}</th>
                <th style="width: 40%">{{ $t('plugin_opencode.path') }}</th>
                <th style="width: 60px" class="text-center">{{ $t('plugin_opencode.action') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-if="!fsLoading && fsItems.length === 0">
                <td colspan="3" class="text-center text-medium-emphasis">{{ $t('plugin_opencode.no_entries') }}</td>
              </tr>
              <tr
                v-for="item in fsItems"
                :key="item.path"
                :class="['cursor-pointer', fsActiveItem && fsActiveItem.path === item.path ? 'bg-primary bg-opacity-10' : '']"
                @click="fsActiveItem = item"
                @dblclick.stop.prevent="fsNavigateInto(item)"
              >
                <td>
                  <div class="d-flex align-center ga-2">
                    <v-icon size="18">mdi-folder</v-icon>
                    <span>{{ item.name }}</span>
                  </div>
                </td>
                <td><span class="text-caption">{{ item.displayPath || item.path }}</span></td>
                <td class="text-center">
                  <v-btn size="small" icon="mdi-folder-open" variant="text" @click.stop="fsNavigateInto(item)" :disabled="fsLoading" />
                </td>
              </tr>
            </tbody>
          </v-table>
        </v-card-text>
        <v-divider />
        <v-card-actions class="d-flex align-center">
          <div class="text-caption text-truncate" style="max-width: 60%">
            <strong>{{ $t('plugin_opencode.selected') }}:</strong>
            <span v-if="fsActiveItem">{{ fsActiveItem.displayPath || fsActiveItem.path }}</span>
            <span v-else>-</span>
          </div>
          <v-spacer />
          <v-btn variant="text" color="onPrimary" @click="fsDialog = false">{{ $t('plugin_opencode.cancel') }}</v-btn>
          <v-btn color="onPrimary" @click="fsSelect" :disabled="fsLoading">{{ $t('plugin_opencode.select') }}</v-btn>
        </v-card-actions>
      </v-card>
    </v-dialog>
  </div>
</template>

<script setup>
import { ref, reactive, computed, onMounted, onUnmounted, nextTick } from 'vue';
import { Terminal } from '@xterm/xterm';
import { FitAddon } from '@xterm/addon-fit';
import { ClipboardAddon } from '@xterm/addon-clipboard';
import { io } from 'socket.io-client';
import '@xterm/xterm/css/xterm.css';

const PLUGIN_NAME = 'opencode';

const loading = ref(true);
const saving = ref(false);
const overlay = ref(false);
const starting = ref(false);
const stopping = ref(false);
const checkingUpdates = ref(false);
const updating = ref(false);
const installing = ref(false);
const statusInterval = ref(null);
const fsDialog = ref(false);
const fsCurrentPath = ref('');
const fsItems = ref([]);
const fsLoading = ref(false);
const fsActiveItem = ref(null);

const running = ref(false);
const currentVersion = ref('');
const latestVersion = ref('');
const updateAvailable = ref(false);

const settings = reactive({
  auto_start: false,
  hostname: '0.0.0.0',
  port: '4096',
  data_home: '',
  version: ''
});

const dataHomeSaved = ref(false);
const recommendedPath = ref('');
const terminalContainer = ref(null);
const terminalActive = ref(false);
let term = null;
let fitAddon = null;
let clipboardAddon = null;
let socket = null;
let terminalSessionId = null;
let resizeHandler = null;

const webuiEnabled = computed(() => {
  const port = parseInt(settings.port, 10);
  return port > 0;
});

const webuiUrl = computed(() => {
  const port = settings.port || '4096';
  return `${window.location.protocol}//${window.location.hostname}:${port}`;
});

const getAuthHeaders = () => ({
  Authorization: 'Bearer ' + localStorage.getItem('authToken'),
});

const getDataHome = async () => {
  let path = settings.data_home;
  if (!path) {
    try {
      const res = await fetch('/api/v1/mos/settings/docker', {
        headers: getAuthHeaders(),
      });
      if (res.ok) {
        const data = await res.json();
        if (data.appdata) {
          path = data.appdata + '/opencode';
        }
      }
    } catch (e) {
      console.error('Failed to fetch docker settings:', e);
    }
  }
  return path;
};

const doInstall = async () => {
  installing.value = true;
  try {
    const dataHome = settings.data_home;

    if (!dataHome) {
      alert('Error: Data home not set. Please set data home and save first.');
      installing.value = false;
      return;
    }

    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        data_home: dataHome,
        hostname: settings.hostname,
        port: settings.port,
        auto_start: settings.auto_start,
        version: '',
      }),
    });

    dataHomeSaved.value = true;
    recommendedPath.value = '';
    currentVersion.value = '';

    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Install failed');
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to install:', e);
    alert('Failed to install OpenCode. Check logs for details.');
  } finally {
    installing.value = false;
  }
};

const doUpdate = async () => {
  updating.value = true;
  try {
    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'opencode',
          args: ['stop'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['install_binary'],
        timeout: 120,
        parse_json: false,
      }),
    });

    if (!res.ok) {
      throw new Error('Update failed');
    }

    if (running.value) {
      await fetch('/api/v1/mos/plugins/query', {
        method: 'POST',
        headers: {
          ...getAuthHeaders(),
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          command: 'opencode',
          args: ['start'],
          timeout: 30,
          parse_json: false,
        }),
      });
    }

    await fetchSettings();
    await checkStatus();
    await checkForUpdates();
  } catch (e) {
    console.error('Failed to update:', e);
    alert('Failed to update OpenCode. Check logs for details.');
  } finally {
    updating.value = false;
  }
};

const openFsDialog = async () => {
  fsCurrentPath.value = settings.data_home || '/mnt';
  fsDialog.value = true;
  await fetchDirectory(fsCurrentPath.value);
};

const fetchDirectory = async (dirPath) => {
  fsLoading.value = true;
  fsActiveItem.value = null;
  try {
    const url = new URL('/api/v1/mos/fsnavigator', window.location.origin);
    if (dirPath && dirPath !== '/') {
      url.searchParams.set('path', dirPath);
    }
    url.searchParams.set('type', 'directories');
    url.searchParams.set('roots', '/mnt');

    const res = await fetch(url.toString(), {
      headers: getAuthHeaders(),
    });

    if (res.ok) {
      const data = await res.json();
      fsCurrentPath.value = data.currentPath || dirPath || '/mnt';
      fsItems.value = Array.isArray(data.items) ? data.items : [];
    } else {
      fsItems.value = [];
    }
  } catch (e) {
    console.error('Failed to fetch directory:', e);
    fsItems.value = [];
  } finally {
    fsLoading.value = false;
  }
};

const fsNavigateInto = (item) => {
  if (!item || item.type !== 'directory') return;
  fetchDirectory(item.path);
};

const fsNavigateUp = () => {
  if (!fsCurrentPath.value || fsCurrentPath.value === '/mnt') return;
  const parts = fsCurrentPath.value.split('/').filter(Boolean);
  parts.pop();
  const parentPath = '/' + parts.join('/');
  fetchDirectory(parentPath || '/mnt');
};

const fsGoRoot = () => {
  fetchDirectory('/mnt');
};

const fsSelect = () => {
  if (fsActiveItem.value) {
    settings.data_home = fsActiveItem.value.path;
  } else {
    settings.data_home = fsCurrentPath.value;
  }
  fsDialog.value = false;
};

const fetchSettings = async () => {
  try {
    const res = await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      headers: getAuthHeaders(),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.data_home !== undefined && data.data_home) {
        settings.data_home = data.data_home;
        dataHomeSaved.value = true;
        recommendedPath.value = '';
      } else {
        settings.data_home = '';
        dataHomeSaved.value = false;
      }
      if (data.hostname !== undefined) {
        settings.hostname = data.hostname;
      }
      if (data.port !== undefined) {
        settings.port = data.port;
      }
      if (data.auto_start !== undefined) {
        settings.auto_start = data.auto_start;
      }
      if (data.version) {
        currentVersion.value = data.version;
      }
    }

    if (!dataHomeSaved.value && !settings.data_home) {
      try {
        const dockerRes = await fetch('/api/v1/mos/settings/docker', {
          headers: getAuthHeaders(),
        });
        if (dockerRes.ok) {
          const dockerData = await dockerRes.json();
          if (dockerData.appdata) {
            recommendedPath.value = dockerData.appdata + '/opencode';
          }
        }
      } catch (e) {
        console.error('Failed to fetch docker settings:', e);
      }
    }
  } catch (e) {
    console.error('Failed to fetch settings:', e);
  }
};

const checkStatus = async () => {
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['status'],
        timeout: 5,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        running.value = data.output.running === true;
      }
    }
  } catch (e) {
    running.value = false;
  }
};

const checkForUpdates = async () => {
  checkingUpdates.value = true;
  try {
    const res = await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['check_version'],
        timeout: 15,
        parse_json: true,
      }),
    });
    if (res.ok) {
      const data = await res.json();
      if (data.success && data.output) {
        latestVersion.value = data.output.latest || '';
        updateAvailable.value = currentVersion.value && data.output.update_available === true;
      }
    }
  } catch (e) {
    console.error('Failed to check updates:', e);
  } finally {
    checkingUpdates.value = false;
  }
};

const startDaemon = async () => {
  starting.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['start'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to start:', e);
  } finally {
    starting.value = false;
  }
};

const stopDaemon = async () => {
  stopping.value = true;
  try {
    await fetch('/api/v1/mos/plugins/query', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: 'opencode',
        args: ['stop'],
        timeout: 30,
        parse_json: false,
      }),
    });
    await checkStatus();
  } catch (e) {
    console.error('Failed to stop:', e);
  } finally {
    stopping.value = false;
  }
};

const saveSettings = async () => {
  saving.value = true;
  try {
    await fetch(`/api/v1/mos/plugins/settings/${PLUGIN_NAME}`, {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        auto_start: settings.auto_start,
        hostname: settings.hostname,
        port: settings.port,
        data_home: settings.data_home,
        version: currentVersion.value,
      }),
    });
    dataHomeSaved.value = true;
    recommendedPath.value = '';
  } catch (e) {
    console.error('Failed to save settings:', e);
  } finally {
    saving.value = false;
  }
};

const openTerminal = async () => {
  terminalActive.value = true;
  await nextTick();

  try {
    const res = await fetch('/api/v1/terminal/create', {
      method: 'POST',
      headers: {
        ...getAuthHeaders(),
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        command: '/usr/bin/opencode',
        args: [],
      }),
    });

    if (!res.ok) {
      throw new Error('Failed to create terminal session');
    }

    const session = await res.json();
    terminalSessionId = session.sessionId;

    fitAddon = new FitAddon();
    clipboardAddon = new ClipboardAddon();

    term = new Terminal({ cursorBlink: true, fontFamily: 'monospace', fontSize: 14 });
    term.loadAddon(fitAddon);
    term.loadAddon(clipboardAddon);
    term.open(terminalContainer.value);
    term.focus();
    fitAddon.fit();

    term.onSelectionChange(() => {
      if (!term.hasSelection()) return;
      const selected = term.getSelection();

      if (navigator.clipboard && window.isSecureContext) {
        navigator.clipboard.writeText(selected);
      } else {
        const textarea = document.createElement('textarea');
        textarea.value = selected;
        textarea.style.cssText = 'position:fixed;opacity:0';
        document.body.appendChild(textarea);
        textarea.focus();
        textarea.select();
        try {
          document.execCommand('copy');
        } catch (e) {}
        document.body.removeChild(textarea);
      }
    });

    socket = io('/terminal', { path: '/api/v1/socket.io/' });

    socket.on('connect', () => {
      socket.emit('join-session', {
        sessionId: terminalSessionId,
        token: localStorage.getItem('authToken'),
      });
    });

    socket.on('session-joined', () => {
      fitAddon.fit();
      socket.emit('terminal-resize', { cols: term.cols, rows: term.rows });
    });

    socket.on('terminal-output', (data) => {
      term.write(data);
    });

    term.onData((data) => {
      socket.emit('terminal-input', data);
    });

    term.onResize(({ cols, rows }) => {
      socket.emit('terminal-resize', { cols, rows });
    });

    resizeHandler = () => {
      if (fitAddon) fitAddon.fit();
    };
    window.addEventListener('resize', resizeHandler);

    socket.on('disconnect', () => {
      if (term) term.write('\r\nConnection closed.\r\n');
    });
  } catch (e) {
    console.error('Failed to open terminal:', e);
    terminalActive.value = false;
  }
};

const closeTerminal = () => {
  if (resizeHandler) {
    window.removeEventListener('resize', resizeHandler);
    resizeHandler = null;
  }
  if (socket) {
    socket.emit('leave-session');
    socket.disconnect();
    socket = null;
  }
  if (term) {
    term.dispose();
    term = null;
  }
  if (fitAddon) {
    fitAddon.dispose();
    fitAddon = null;
  }
  if (clipboardAddon) {
    clipboardAddon.dispose();
    clipboardAddon = null;
  }
  terminalSessionId = null;
  terminalActive.value = false;
};

onMounted(async () => {
  try {
    await fetchSettings();
    await checkStatus();
    if (!currentVersion.value) {
      checkForUpdates();
    }

    if (webuiEnabled.value) {
      statusInterval.value = setInterval(async () => {
        await checkStatus();
      }, 5000);
    }
  } catch (e) {
    console.error('Failed to initialize:', e);
  } finally {
    loading.value = false;
  }
});

onUnmounted(() => {
  if (statusInterval.value) {
    clearInterval(statusInterval.value);
  }
  closeTerminal();
});
</script>
