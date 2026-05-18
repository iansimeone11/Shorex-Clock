const STORAGE_KEY = "shorex-clock-data-v1";
const SESSION_KEY = "shorex-clock-session-v1";
const MODE_KEY = "shorex-clock-device-mode-v1";
const ADMIN_PIN = "1234";
const memoryStore = {};
const localStore = getStorage("localStorage");
const sessionStore = getStorage("sessionStorage");

function getStorage(name) {
  try {
    return globalThis[name];
  } catch {
    return null;
  }
}

function storageGet(storage, key) {
  try {
    return storage.getItem(key);
  } catch {
    return memoryStore[key] || null;
  }
}

function storageSet(storage, key, value) {
  try {
    storage.setItem(key, value);
  } catch {
    memoryStore[key] = value;
  }
}

function storageRemove(storage, key) {
  try {
    storage.removeItem(key);
  } catch {
    delete memoryStore[key];
  }
}

function createId() {
  if (globalThis.crypto?.randomUUID) {
    return globalThis.crypto.randomUUID();
  }

  return `id-${Date.now()}-${Math.random().toString(16).slice(2)}`;
}

const initialState = {
  people: [
    { id: createId(), name: "Ian", rate: 0, pin: "1234" },
    { id: createId(), name: "Equipo Shorex", rate: 0, pin: "1234" },
  ],
  shifts: [],
};

let state = loadState();
let selectedPersonId = state.people[0]?.id || "";
let session = loadSession();

const els = {
  authShell: document.querySelector("#authShell"),
  appShell: document.querySelector("#appShell"),
  loginModeButton: document.querySelector("#loginModeButton"),
  registerModeButton: document.querySelector("#registerModeButton"),
  loginForm: document.querySelector("#loginForm"),
  registerForm: document.querySelector("#registerForm"),
  loginButton: document.querySelector("#loginButton"),
  registerButton: document.querySelector("#registerButton"),
  loginUser: document.querySelector("#loginUser"),
  loginPin: document.querySelector("#loginPin"),
  loginMessage: document.querySelector("#loginMessage"),
  registerName: document.querySelector("#registerName"),
  registerPin: document.querySelector("#registerPin"),
  registerMessage: document.querySelector("#registerMessage"),
  authLoginToggle: document.querySelector("#authLoginToggle"),
  authRegisterToggle: document.querySelector("#authRegisterToggle"),
  mobileModeButton: document.querySelector("#mobileModeButton"),
  desktopModeButton: document.querySelector("#desktopModeButton"),
  adminLoginButton: document.querySelector("#adminLoginButton"),
  currentUserLabel: document.querySelector("#currentUserLabel"),
  logoutButton: document.querySelector("#logoutButton"),
  todayLabel: document.querySelector("#todayLabel"),
  timeLabel: document.querySelector("#timeLabel"),
  statusLabel: document.querySelector("#statusLabel"),
  viewTabs: document.querySelector("#viewTabs"),
  employeeTab: document.querySelector("#employeeTab"),
  adminTab: document.querySelector("#adminTab"),
  employeeView: document.querySelector("#employeeView"),
  adminView: document.querySelector("#adminView"),
  locationSelect: document.querySelector("#locationSelect"),
  clientSelect: document.querySelector("#clientSelect"),
  locationButtons: document.querySelector("#locationButtons"),
  clientButtons: document.querySelector("#clientButtons"),
  clockButton: document.querySelector("#clockButton"),
  sessionHelper: document.querySelector("#sessionHelper"),
  todayHours: document.querySelector("#todayHours"),
  weekHours: document.querySelector("#weekHours"),
  employeeHistory: document.querySelector("#employeeHistory"),
  fromDate: document.querySelector("#fromDate"),
  toDate: document.querySelector("#toDate"),
  exportButton: document.querySelector("#exportButton"),
  clearButton: document.querySelector("#clearButton"),
  summaryGrid: document.querySelector("#summaryGrid"),
  ratesList: document.querySelector("#ratesList"),
  confirmOverlay: document.querySelector("#confirmOverlay"),
  confirmTitle: document.querySelector("#confirmTitle"),
  confirmCopy: document.querySelector("#confirmCopy"),
  cancelClockButton: document.querySelector("#cancelClockButton"),
  confirmClockButton: document.querySelector("#confirmClockButton"),
};

let pendingClockAction = null;
let remoteApi = null;
let deviceMode = storageGet(localStore, MODE_KEY) || "mobile";

function loadState() {
  const raw = storageGet(localStore, STORAGE_KEY);
  if (!raw) return initialState;

  try {
    const saved = JSON.parse(raw);
    return {
      people: normalizePeople(Array.isArray(saved.people) ? saved.people : initialState.people),
      shifts: Array.isArray(saved.shifts) ? saved.shifts : [],
    };
  } catch {
    return initialState;
  }
}

function normalizePeople(people) {
  return people.map((person) => ({
    ...person,
    id: person.id || createId(),
    pin: person.pin || "1234",
    rate: Number(person.rate || 0),
  }));
}

function saveState() {
  storageSet(localStore, STORAGE_KEY, JSON.stringify(state));
}

function loadSession() {
  const raw = storageGet(sessionStore, SESSION_KEY) || storageGet(localStore, SESSION_KEY);
  if (!raw) return null;

  try {
    return JSON.parse(raw);
  } catch {
    return null;
  }
}

function saveSession(nextSession) {
  session = nextSession;
  if (nextSession) {
    storageSet(sessionStore, SESSION_KEY, JSON.stringify(nextSession));
    storageSet(localStore, SESSION_KEY, JSON.stringify(nextSession));
  } else {
    storageRemove(sessionStore, SESSION_KEY);
    storageRemove(localStore, SESSION_KEY);
  }
}

async function loadRemoteConfig() {
  if (location.protocol === "file:") return null;

  try {
    const response = await fetch("/api/config", { cache: "no-store" });
    if (!response.ok) return null;

    const config = await response.json();
    if (!config.supabaseUrl || !config.supabaseAnonKey) return null;
    return config;
  } catch {
    return null;
  }
}

function createRemoteApi(config) {
  async function rpc(name, body) {
    const response = await fetch(`${config.supabaseUrl}/rest/v1/rpc/${name}`, {
      method: "POST",
      headers: {
        apikey: config.supabaseAnonKey,
        Authorization: `Bearer ${config.supabaseAnonKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify(body),
    });

    if (!response.ok) {
      throw new Error("No se pudo conectar con la base de datos.");
    }

    const data = await response.json();
    if (data && data.ok === false) {
      throw new Error(data.message || "La operación no se pudo completar.");
    }

    return data;
  }

  return {
    register: (name, pin) => rpc("app_register_employee", { p_name: name, p_pin: pin }),
    login: (name, pin) => rpc("app_login_employee", { p_name: name, p_pin: pin }),
    employeeState: (personId) => rpc("app_employee_state", { p_employee_id: personId }),
    clockIn: (personId, locationName, clientName) =>
      rpc("app_clock_in", { p_employee_id: personId, p_location: locationName, p_client: clientName }),
    clockOut: (personId) => rpc("app_clock_out", { p_employee_id: personId }),
    adminLogin: (pin) => rpc("app_admin_login", { p_pin: pin }),
    adminState: (token) => rpc("app_admin_state", { p_token: token }),
    updateRate: (token, personId, rate) => rpc("app_update_rate", { p_token: token, p_employee_id: personId, p_rate: rate }),
    clearShifts: (token) => rpc("app_clear_shifts", { p_token: token }),
  };
}

function applyRemoteState(data) {
  state = {
    people: normalizePeople(data.people || []),
    shifts: (data.shifts || []).map((shift) => ({
      id: shift.id,
      personId: shift.personId,
      location: shift.location,
      client: shift.client,
      clockIn: shift.clockIn,
      clockOut: shift.clockOut,
    })),
  };

  if (session?.role === "employee") {
    selectedPersonId = session.personId;
  } else if (!state.people.some((person) => person.id === selectedPersonId)) {
    selectedPersonId = state.people[0]?.id || "";
  }
}

async function refreshRemoteSession() {
  if (!remoteApi || session?.source !== "supabase") return;

  const data = session.role === "admin"
    ? await remoteApi.adminState(session.adminToken)
    : await remoteApi.employeeState(session.personId);
  applyRemoteState(data);
}

function normalizeName(value) {
  return value.trim().replace(/\s+/g, " ").toLowerCase();
}

function firstName(name) {
  return name.trim().split(/\s+/)[0] || name;
}

function welcomeLabel(name) {
  const first = firstName(name);
  const normalized = normalizeName(first).normalize("NFD").replace(/[\u0300-\u036f]/g, "");
  const femaleNames = new Set([
    "ana",
    "andrea",
    "camila",
    "carla",
    "carolina",
    "catalina",
    "cecilia",
    "claudia",
    "daniela",
    "elena",
    "florencia",
    "gabriela",
    "julia",
    "julieta",
    "laura",
    "lucia",
    "luisa",
    "maria",
    "mariana",
    "martina",
    "natalia",
    "paula",
    "romina",
    "sabrina",
    "sofia",
    "valentina",
    "victoria",
  ]);
  const isLikelyFemale = femaleNames.has(normalized) || normalized.endsWith("a");

  return `${isLikelyFemale ? "Bienvenida" : "Bienvenido"} ${first}!`;
}

function nowIso() {
  return new Date().toISOString();
}

function formatDateTime(iso) {
  return new Intl.DateTimeFormat("es-AR", {
    dateStyle: "short",
    timeStyle: "short",
  }).format(new Date(iso));
}

function formatTime(date) {
  return new Intl.DateTimeFormat("es-AR", {
    hour: "2-digit",
    minute: "2-digit",
    hour12: false,
  }).format(date);
}

function formatDay(date) {
  return new Intl.DateTimeFormat("es-AR", {
    weekday: "long",
    day: "numeric",
    month: "long",
  }).format(date);
}

function formatHours(hours) {
  return `${hours.toFixed(2)} h`;
}

function formatMoney(value) {
  return new Intl.NumberFormat("es-AR", {
    style: "currency",
    currency: "ARS",
    maximumFractionDigits: 0,
  }).format(value);
}

function startOfDay(date) {
  return new Date(date.getFullYear(), date.getMonth(), date.getDate());
}

function endOfDay(date) {
  return new Date(date.getFullYear(), date.getMonth(), date.getDate(), 23, 59, 59, 999);
}

function startOfWeek(date) {
  const day = date.getDay() || 7;
  const start = startOfDay(date);
  start.setDate(start.getDate() - day + 1);
  return start;
}

function minutesBetween(startIso, endIso = nowIso()) {
  return Math.max(0, new Date(endIso) - new Date(startIso)) / 60000;
}

function shiftHours(shift) {
  return minutesBetween(shift.clockIn, shift.clockOut) / 60;
}

function activeShift(personId) {
  return state.shifts.find((shift) => shift.personId === personId && !shift.clockOut);
}

function shiftsForPerson(personId) {
  return state.shifts
    .filter((shift) => shift.personId === personId)
    .sort((a, b) => new Date(b.clockIn) - new Date(a.clockIn));
}

function shiftsInRange(shifts) {
  const from = els.fromDate.value ? startOfDay(new Date(`${els.fromDate.value}T00:00:00`)) : null;
  const to = els.toDate.value ? endOfDay(new Date(`${els.toDate.value}T00:00:00`)) : null;

  return shifts.filter((shift) => {
    const started = new Date(shift.clockIn);
    return (!from || started >= from) && (!to || started <= to);
  });
}

function sumHours(shifts) {
  return shifts.reduce((total, shift) => total + shiftHours(shift), 0);
}

function renderClock() {
  const now = new Date();
  els.todayLabel.textContent = formatDay(now);
  els.timeLabel.textContent = formatTime(now);
}

function renderPeople() {
  if (!state.people.some((person) => person.id === selectedPersonId)) {
    selectedPersonId = state.people[0]?.id || "";
  }
}

function renderEmployee() {
  const person = state.people.find((item) => item.id === selectedPersonId);
  const active = person ? activeShift(person.id) : null;
  const todayStart = startOfDay(new Date());
  const weekStart = startOfWeek(new Date());
  const personShifts = person ? shiftsForPerson(person.id) : [];

  els.clockButton.disabled = !person;
  els.locationSelect.disabled = Boolean(active);
  els.clientSelect.disabled = Boolean(active);
  syncChoiceButtons();
  els.clockButton.textContent = active ? "Clock out" : "Clock in";
  els.statusLabel.textContent = active ? `${person.name} está trabajando` : person ? `${person.name} está fuera` : "Elegí tu usuario";
  els.sessionHelper.textContent = active
    ? `Entrada: ${formatDateTime(active.clockIn)} · ${active.location || "Sin ubicacion"} · ${active.client || "Sin cliente"}`
    : person
      ? "Elegí ubicación y cliente antes de hacer clock in."
      : "Agregá una persona para empezar.";

  els.todayHours.textContent = formatHours(sumHours(personShifts.filter((shift) => new Date(shift.clockIn) >= todayStart)));
  els.weekHours.textContent = formatHours(sumHours(personShifts.filter((shift) => new Date(shift.clockIn) >= weekStart)));

  els.employeeHistory.innerHTML = "";
  if (!personShifts.length) {
    els.employeeHistory.innerHTML = '<p class="empty">Todavía no hay registros.</p>';
    return;
  }

  personShifts.slice(0, 6).forEach((shift) => {
    const item = document.createElement("div");
    item.className = "history-item";
    item.innerHTML = `
      <div>
        <strong>${formatDateTime(shift.clockIn)}</strong>
        <span>${shift.clockOut ? `Salida: ${formatDateTime(shift.clockOut)}` : "Turno abierto"}</span>
        <span>${shift.location || "Sin ubicacion"} · ${shift.client || "Sin cliente"}</span>
      </div>
      <strong>${formatHours(shiftHours(shift))}</strong>
    `;
    els.employeeHistory.appendChild(item);
  });
}

function syncChoiceButtons() {
  const active = selectedPersonId ? activeShift(selectedPersonId) : null;
  const disableChoices = Boolean(active);

  els.locationButtons.querySelectorAll("[data-location]").forEach((button) => {
    button.classList.toggle("is-selected", button.dataset.location === els.locationSelect.value);
    button.disabled = disableChoices;
  });

  els.clientButtons.querySelectorAll("[data-client]").forEach((button) => {
    button.classList.toggle("is-selected", button.dataset.client === els.clientSelect.value);
    button.disabled = disableChoices;
  });
}

function renderSession() {
  const isLoggedIn = Boolean(session);
  els.authShell.classList.toggle("is-hidden", isLoggedIn);
  els.appShell.classList.toggle("is-hidden", !isLoggedIn);

  if (!isLoggedIn) return;

  const person = state.people.find((item) => item.id === session.personId);
  if (session.role === "admin") {
    els.currentUserLabel.textContent = "Bienvenido Admin!";
    els.viewTabs.classList.remove("is-hidden");
    els.employeeTab.classList.add("is-hidden");
    els.adminTab.classList.remove("is-hidden");
    switchView("admin");
    return;
  }

  selectedPersonId = person?.id || selectedPersonId;
  els.currentUserLabel.textContent = person ? welcomeLabel(person.name) : "Bienvenido!";
  els.viewTabs.classList.add("is-hidden");
  els.adminTab.classList.add("is-hidden");
  switchView("employee");
}

function renderAdmin(options = { rates: true }) {
  const filteredShifts = shiftsInRange(state.shifts);
  els.summaryGrid.innerHTML = "";

  if (!state.people.length) {
    els.summaryGrid.innerHTML = '<p class="empty">No hay personas cargadas.</p>';
  }

  state.people.forEach((person) => {
    const personShifts = filteredShifts.filter((shift) => shift.personId === person.id);
    const total = sumHours(personShifts);
    const open = personShifts.filter((shift) => !shift.clockOut).length;
    const card = document.createElement("article");
    card.className = "summary-card";
    card.innerHTML = `
      <h3>${person.name}</h3>
      <dl>
        <dt>Horas</dt>
        <dd>${formatHours(total)}</dd>
        <dt>Turnos</dt>
        <dd>${personShifts.length}</dd>
        <dt>Abiertos</dt>
        <dd>${open}</dd>
        <dt>Pago estimado</dt>
        <dd>${formatMoney(total * Number(person.rate || 0))}</dd>
      </dl>
    `;
    els.summaryGrid.appendChild(card);
  });

  if (!options.rates) return;

  els.ratesList.innerHTML = "";
  state.people.forEach((person) => {
    const row = document.createElement("div");
    row.className = "rate-row";
    row.innerHTML = `
      <div>
        <strong>${person.name}</strong>
        <span>Valor por hora</span>
      </div>
      <input type="number" min="0" step="100" value="${person.rate || 0}" data-rate-id="${person.id}" aria-label="Tarifa de ${person.name}" />
    `;
    els.ratesList.appendChild(row);
  });
}

function renderAll() {
  renderPeople();
  renderSession();
  renderEmployee();
  renderAdmin();
}

function switchView(view) {
  if (view === "admin" && session?.role !== "admin") return;

  const isAdmin = view === "admin";
  els.employeeTab.classList.toggle("is-active", !isAdmin);
  els.adminTab.classList.toggle("is-active", isAdmin);
  els.employeeView.classList.toggle("is-active", !isAdmin);
  els.adminView.classList.toggle("is-active", isAdmin);
}

function setDefaultDateRange() {
  const today = new Date();
  const weekStart = startOfWeek(today);
  els.fromDate.valueAsDate = weekStart;
  els.toDate.valueAsDate = today;
}

function downloadCsv() {
  const rows = [["Persona", "Ubicacion", "Cliente", "Entrada", "Salida", "Horas", "Tarifa", "Pago"]];
  const filteredShifts = shiftsInRange(state.shifts);

  filteredShifts.forEach((shift) => {
    const person = state.people.find((item) => item.id === shift.personId);
    const rate = Number(person?.rate || 0);
    const hours = shiftHours(shift);
    rows.push([
      person?.name || "Sin persona",
      shift.location || "",
      shift.client || "",
      formatDateTime(shift.clockIn),
      shift.clockOut ? formatDateTime(shift.clockOut) : "Abierto",
      hours.toFixed(2),
      rate.toString(),
      (hours * rate).toFixed(2),
    ]);
  });

  const csv = rows
    .map((row) => row.map((cell) => `"${String(cell).replaceAll('"', '""')}"`).join(","))
    .join("\n");
  const blob = new Blob([csv], { type: "text/csv;charset=utf-8" });
  const url = URL.createObjectURL(blob);
  const link = document.createElement("a");
  link.href = url;
  link.download = "shorex-horas.csv";
  link.click();
  URL.revokeObjectURL(url);
}

function showClockConfirmation(action, shiftData = null) {
  pendingClockAction = { action, shiftData };
  const isClockIn = action === "in";

  els.confirmTitle.textContent = isClockIn
    ? "¿Está seguro que quiere hacer clock-in?"
    : "¿Está seguro que quiere hacer clock-out?";
  els.confirmCopy.textContent = isClockIn
    ? `Se va a registrar la entrada en ${shiftData.location} para ${shiftData.client}. Recordá que el clock-in debe realizarse una vez uniformado y en posición, listo para iniciar el servicio.`
    : "Se va a cerrar el turno abierto y calcular las horas trabajadas.";
  els.confirmClockButton.textContent = isClockIn ? "Confirmar clock-in" : "Confirmar clock-out";
  els.confirmOverlay.classList.remove("is-hidden");
}

function hideClockConfirmation() {
  pendingClockAction = null;
  els.confirmOverlay.classList.add("is-hidden");
}

function commitClockAction() {
  if (!pendingClockAction || !selectedPersonId) return;

  if (remoteApi && session?.source === "supabase") {
    const action = pendingClockAction;
    hideClockConfirmation();

    const request = action.action === "out"
      ? remoteApi.clockOut(selectedPersonId)
      : remoteApi.clockIn(selectedPersonId, action.shiftData.location, action.shiftData.client);

    request
      .then((data) => {
        applyRemoteState(data);
        renderAll();
      })
      .catch((error) => {
        els.sessionHelper.textContent = error.message;
      });
    return;
  }

  const active = activeShift(selectedPersonId);

  if (pendingClockAction.action === "out") {
    if (active) {
      active.clockOut = nowIso();
    }
  } else if (pendingClockAction.shiftData) {
    state.shifts.push({
      id: createId(),
      personId: selectedPersonId,
      location: pendingClockAction.shiftData.location,
      client: pendingClockAction.shiftData.client,
      clockIn: nowIso(),
      clockOut: null,
    });
  }

  hideClockConfirmation();
  saveState();
  renderAll();
}

function switchAuthMode(mode) {
  const isRegister = mode === "register";
  els.authLoginToggle.checked = !isRegister;
  els.authRegisterToggle.checked = isRegister;
  els.loginModeButton.classList.toggle("is-active", !isRegister);
  els.registerModeButton.classList.toggle("is-active", isRegister);
  els.loginForm.classList.toggle("is-active", !isRegister);
  els.registerForm.classList.toggle("is-active", isRegister);
  els.loginMessage.textContent = "";
  els.registerMessage.textContent = "";
}

function setDeviceMode(mode) {
  deviceMode = mode === "desktop" ? "desktop" : "mobile";
  document.body.classList.toggle("app-mode-desktop", deviceMode === "desktop");
  document.body.classList.toggle("app-mode-mobile", deviceMode === "mobile");
  els.mobileModeButton.classList.toggle("is-active", deviceMode === "mobile");
  els.desktopModeButton.classList.toggle("is-active", deviceMode === "desktop");
  storageSet(localStore, MODE_KEY, deviceMode);
}

els.loginModeButton.addEventListener("click", () => switchAuthMode("login"));
els.registerModeButton.addEventListener("click", () => switchAuthMode("register"));
els.mobileModeButton.addEventListener("click", () => setDeviceMode("mobile"));
els.desktopModeButton.addEventListener("click", () => setDeviceMode("desktop"));

async function handleLogin(event) {
  event.preventDefault();
  const username = normalizeName(els.loginUser.value);
  const pin = els.loginPin.value.trim();

  if (remoteApi) {
    try {
      const data = await remoteApi.login(username, pin);
      const person = data.people?.[0];
      if (!person) throw new Error("Usuario o PIN incorrecto.");

      selectedPersonId = person.id;
      saveSession({ source: "supabase", role: "employee", personId: person.id });
      applyRemoteState(data);
      els.loginUser.value = "";
      els.loginPin.value = "";
      renderAll();
    } catch (error) {
      els.loginMessage.textContent = error.message;
    }
    return;
  }

  const person = state.people.find((item) => normalizeName(item.name) === username);
  if (!person || person.pin !== pin) {
    els.loginMessage.textContent = "Usuario o PIN incorrecto.";
    return;
  }

  selectedPersonId = person.id;
  saveSession({ role: "employee", personId: person.id });
  els.loginUser.value = "";
  els.loginPin.value = "";
  renderAll();
}

async function handleRegister(event) {
  event.preventDefault();
  const name = els.registerName.value.trim();
  const pin = els.registerPin.value.trim();

  if (name.length < 2) {
    els.registerMessage.textContent = "Escribi tu nombre.";
    return;
  }

  if (pin.length < 4) {
    els.registerMessage.textContent = "El PIN necesita minimo 4 digitos.";
    return;
  }

  if (remoteApi) {
    try {
      const data = await remoteApi.register(name, pin);
      const person = data.people?.[0];
      if (!person) throw new Error("No se pudo crear el usuario.");

      selectedPersonId = person.id;
      saveSession({ source: "supabase", role: "employee", personId: person.id });
      applyRemoteState(data);
      els.registerName.value = "";
      els.registerPin.value = "";
      renderAll();
    } catch (error) {
      els.registerMessage.textContent = error.message;
    }
    return;
  }

  const person = { id: createId(), name, rate: 0, pin };
  state.people.push(person);
  selectedPersonId = person.id;
  saveState();
  saveSession({ role: "employee", personId: person.id });
  els.registerName.value = "";
  els.registerPin.value = "";
  renderAll();
}

els.loginForm.addEventListener("submit", handleLogin);
els.loginButton.addEventListener("click", handleLogin);
els.registerForm.addEventListener("submit", handleRegister);
els.registerButton.addEventListener("click", handleRegister);

els.adminLoginButton.addEventListener("click", () => {
  const pin = window.prompt("PIN de admin");
  if (remoteApi) {
    remoteApi.adminLogin(pin)
      .then((loginData) => {
        saveSession({ source: "supabase", role: "admin", adminToken: loginData.token });
        return remoteApi.adminState(loginData.token);
      })
      .then((data) => {
        applyRemoteState(data);
        switchView("admin");
        renderAll();
      })
      .catch((error) => {
        els.loginMessage.textContent = error.message;
      });
    return;
  }

  if (pin !== ADMIN_PIN) {
    els.loginMessage.textContent = "PIN de admin incorrecto.";
    return;
  }

  saveSession({ role: "admin" });
  switchView("admin");
  renderAll();
});

els.logoutButton.addEventListener("click", () => {
  saveSession(null);
  switchAuthMode("login");
  renderAll();
});

els.employeeTab.addEventListener("click", () => switchView("employee"));
els.adminTab.addEventListener("click", () => switchView("admin"));

els.locationSelect.addEventListener("change", syncChoiceButtons);
els.clientSelect.addEventListener("change", syncChoiceButtons);

els.locationButtons.addEventListener("click", (event) => {
  const button = event.target.closest("[data-location]");
  if (!button || button.disabled) return;

  els.locationSelect.value = button.dataset.location;
  syncChoiceButtons();
});

els.clientButtons.addEventListener("click", (event) => {
  const button = event.target.closest("[data-client]");
  if (!button || button.disabled) return;

  els.clientSelect.value = button.dataset.client;
  syncChoiceButtons();
});

els.clockButton.addEventListener("click", () => {
  if (!selectedPersonId) return;
  const active = activeShift(selectedPersonId);

  if (active) {
    showClockConfirmation("out");
  } else {
    const location = els.locationSelect.value;
    const client = els.clientSelect.value;

    if (!location || !client) {
      els.sessionHelper.textContent = "Seleccioná ubicación y cliente antes de hacer clock in.";
      return;
    }

    showClockConfirmation("in", { location, client });
  }
});

els.cancelClockButton.addEventListener("click", hideClockConfirmation);
els.confirmClockButton.addEventListener("click", commitClockAction);
els.confirmOverlay.addEventListener("click", (event) => {
  if (event.target === els.confirmOverlay) {
    hideClockConfirmation();
  }
});

els.fromDate.addEventListener("change", renderAdmin);
els.toDate.addEventListener("change", renderAdmin);
els.exportButton.addEventListener("click", downloadCsv);

els.clearButton.addEventListener("click", () => {
  const confirmed = window.confirm("¿Seguro que querés borrar todos los registros de horas?");
  if (!confirmed) return;

  if (remoteApi && session?.source === "supabase" && session.role === "admin") {
    remoteApi.clearShifts(session.adminToken)
      .then((data) => {
        applyRemoteState(data);
        renderAll();
      })
      .catch((error) => {
        window.alert(error.message);
      });
    return;
  }

  state.shifts = [];
  saveState();
  renderAll();
});

els.ratesList.addEventListener("input", (event) => {
  const input = event.target.closest("[data-rate-id]");
  if (!input) return;

  const person = state.people.find((item) => item.id === input.dataset.rateId);
  if (!person) return;

  person.rate = Number(input.value || 0);

  if (remoteApi && session?.source === "supabase" && session.role === "admin") {
    remoteApi.updateRate(session.adminToken, person.id, person.rate)
      .then((data) => {
        applyRemoteState(data);
        renderAdmin({ rates: false });
      })
      .catch((error) => {
        window.alert(error.message);
      });
    return;
  }

  saveState();
  renderAdmin({ rates: false });
});

async function initializeApp() {
  setDeviceMode(deviceMode);
  setDefaultDateRange();
  renderClock();
  renderAll();

  const config = await loadRemoteConfig();
  if (config) {
    remoteApi = createRemoteApi(config);
    if (session?.source === "supabase") {
      try {
        await refreshRemoteSession();
        renderAll();
      } catch {
        saveSession(null);
        renderAll();
      }
    }
  }

  setInterval(() => {
    renderClock();
    renderEmployee();
    renderAdmin();
  }, 30000);
}

initializeApp();
