// Appli V.1 démo 

import React, { useCallback, useEffect, useMemo, useRef, useState } from "react";
import {
  View,
  Text,
  SafeAreaView,
  Pressable,
  StatusBar,
  FlatList,
  TextInput,
  ScrollView,
  Animated,
  Easing,
  Dimensions,
  Platform,
  Alert,
  Image,
} from "react-native";

import MapView, { Marker, Callout } from "react-native-maps";
import * as Location from "expo-location";
import AsyncStorage from "@react-native-async-storage/async-storage";

import QRCode from "react-native-qrcode-svg";

const { width: SCREEN_W } = Dimensions.get("window");

/* =========================
   ASSETS
========================= */
const LOGO = require("./actifs/logo.png");
const COMPOST_IMG = require("./actifs/compost.png");

/* =========================
   CONFIG
========================= */
const ADMIN = { username: "admin", role: "ADMIN" };
const COMPOST_RATIO = 0.4; // ✅ 1kg déchets -> 0.4kg compost
const MATURATION_DAYS = 20;
const DEFAULT_USER_COORDS = { latitude: 48.8719, longitude: 2.2255 };
const DEFAULT_ADMIN_PIN = "1234";

/* ✅ 1kg = 100 points */
function pointsFromDepositKg(kg) {
  const n = Number(kg);
  if (!Number.isFinite(n) || n <= 0) return 0;
  return Math.round(n * 100);
}

/* =========================
   STORAGE KEYS
========================= */
const K_COMPOSTERS = "greenloop_composters_v3";
const K_DEPOSITS = "greenloop_deposits_v3";
const K_SESSION = "greenloop_session_v3";
const K_PROFILE = "greenloop_profile_v3";
const K_USERS = "greenloop_users_v1";
const K_CURRENT_USER = "greenloop_current_user_v1";
const K_THEME = "greenloop_theme_v3";
const K_ADMIN_PIN = "greenloop_admin_pin_v3";
const K_SPENT_POINTS = "greenloop_spent_points_v1";
const K_REMOTE_CONFIG = "greenloop_remote_config_v1";
const K_LAST_PUSH = "greenloop_last_push_v1";

/* =========================
   THEMES
========================= */
const DARK = {
  mode: "dark",
  bg: "#0B0B0E",
  panel: "#121219",
  card: "#171721",
  card2: "#1C1C27",
  text: "#F4F4F6",
  mutetext: "#B7B7C2",
  line: "rgba(255,255,255,0.08)",
  yellow: "#FFD100",
  red: "#FF2A2A",
  green: "#37D67A",
  shadow: "rgba(0,0,0,0.45)",
  overlay: "rgba(0,0,0,0.55)",
};

const LIGHT = {
  mode: "light",
  bg: "#F6F7F9",
  panel: "#FFFFFF",
  card: "#FFFFFF",
  card2: "#F1F3F6",
  text: "#111114",
  mutetext: "#5A5A66",
  line: "rgba(0,0,0,0.08)",
  yellow: "#FFD100",
  red: "#E62424",
  green: "#2BBF6A",
  shadow: "rgba(0,0,0,0.15)",
  overlay: "rgba(0,0,0,0.35)",
};

/* =========================
   CUVE MODES
========================= */
const VAT_MODES = [
  { key: "REMPLISSAGE", label: "Remplissage", icon: "🧺" },
  { key: "MATURATION", label: "Maturation", icon: "⏳" },
  { key: "PRET", label: "Prêt", icon: "✅" },
];

const MODE_DEFAULT_TEMP = {
  REMPLISSAGE: 40,
  MATURATION: 55,
  PRET: 30,
};

function clampTemp(t) {
  if (!Number.isFinite(t)) return 0;
  return Math.max(-10, Math.min(90, t));
}

/* =========================
   PILOTAGE À DISTANCE
========================= */
const FAN_MODES = [
  { key: "OFF", label: "OFF (laisser chauffer naturellement)" },
  { key: "ON_REG", label: "ON (Régulation : s’active si seuil atteint)" },
  { key: "INTERMITTENT", label: "INTERMITTENT (toutes les X heures)" },
];

const DEFAULT_REMOTE_CONFIG = {
  REMPLISSAGE: {
    tempMin: 25,
    tempMax: 40,
    humidity: 70,
    fanMode: "OFF",
    fanThresholdTemp: 65,
    fanIntervalHours: 2,
    physicalAction: "MÉLANGE : Transfert vers Cuve 2 à J+20.",
  },
  MATURATION: {
    tempMin: 50,
    tempMax: 65,
    humidity: 60,
    fanMode: "ON_REG",
    fanThresholdTemp: 65,
    fanIntervalHours: 2,
    physicalAction: "MÉLANGE : Transfert vers Cuve 3 à J+40.",
  },
  PRET: {
    tempMin: 20,
    tempMax: 30,
    humidity: 50,
    fanMode: "INTERMITTENT",
    fanThresholdTemp: 65,
    fanIntervalHours: 2,
    physicalAction: "RÉCUPÉRATION : Sortie du terreau final à J+60.",
  },
};

/* =========================
   DEMO DATA
========================= */
const SEED_COMPOSTERS = [
  {
    id: "c1",
    name: "Composteur Mairie",
    address: "Centre-ville",
    latitude: 48.8723,
    longitude: 2.2235,
    accepts: ["Épluchures", "Marc de café", "Coquilles d'œufs"],
    vats: [
      { id: "v1", label: "Cuve 1", tempC: 38, mode: "REMPLISSAGE" },
      { id: "v2", label: "Cuve 2", tempC: 52, mode: "MATURATION" },
      { id: "v3", label: "Cuve 3", tempC: 28, mode: "PRET" },
      { id: "v4", label: "Cuve 4 (récup.)", tempC: 26, mode: "PRET" },
    ],
  },
  {
    id: "c2",
    name: "Composteur Parc",
    address: "Entrée principale",
    latitude: 48.8709,
    longitude: 2.2272,
    accepts: ["Déchets verts", "Épluchures", "Serviettes papier"],
    vats: [
      { id: "v1", label: "Cuve 1", tempC: 41, mode: "REMPLISSAGE" },
      { id: "v2", label: "Cuve 2", tempC: 49, mode: "MATURATION" },
      { id: "v3", label: "Cuve 3", tempC: 31, mode: "MATURATION" },
      { id: "v4", label: "Cuve 4 (récup.)", tempC: 26, mode: "PRET" },
    ],
  },
];

/* =========================
   SHOP OFFERS
========================= */
const SHOP_OFFERS = [
  { id: "o1", title: "0,4 kg de compost", kg: 0.4, cost: 100 },
  { id: "o2", title: "1 kg de compost", kg: 1.0, cost: 250 },
  { id: "o3", title: "2 kg de compost", kg: 2.0, cost: 450 },
  { id: "o4", title: "5 kg de compost", kg: 5.0, cost: 900 },
  { id: "o5", title: "10 kg de compost", kg: 10.0, cost: 1700 },
];

/* =========================
   HELPERS
========================= */
function uid(prefix = "id") {
  return `${prefix}_${Math.random().toString(16).slice(2)}_${Date.now().toString(16)}`;
}
function addDays(date, days) {
  const d = new Date(date);
  d.setDate(d.getDate() + days);
  return d;
}
function formatDateFR(d) {
  try {
    return new Intl.DateTimeFormat("fr-FR", { year: "numeric", month: "2-digit", day: "2-digit" }).format(new Date(d));
  } catch {
    const x = new Date(d);
    return `${String(x.getDate()).padStart(2, "0")}/${String(x.getMonth() + 1).padStart(2, "0")}/${x.getFullYear()}`;
  }
}
function kmBetween(a, b) {
  const R = 6371;
  const toRad = (v) => (v * Math.PI) / 180;
  const dLat = toRad(b.latitude - a.latitude);
  const dLon = toRad(b.longitude - a.longitude);
  const lat1 = toRad(a.latitude);
  const lat2 = toRad(b.latitude);
  const s =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(lat1) * Math.cos(lat2) * Math.sin(dLon / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(s));
}
async function readJSON(key, fallback) {
  const raw = await AsyncStorage.getItem(key);
  if (!raw) return fallback;
  try {
    return JSON.parse(raw);
  } catch {
    return fallback;
  }
}
async function writeJSON(key, value) {
  await AsyncStorage.setItem(key, JSON.stringify(value));
}

/* =========================
   QR USERS (LOCAL DEMO)
========================= */
function makeUserId(firstName, lastName) {
  const f = String(firstName || "").trim().toLowerCase();
  const l = String(lastName || "").trim().toLowerCase();
  const base = `${f}_${l}`.replace(/[^a-z0-9_]+/g, "_");
  return base ? `${base}_${Math.random().toString(16).slice(2, 6)}` : uid("user");
}
function buildQrPayload(user) {
  // Format simple, robuste : JSON
  return JSON.stringify({ v: 1, type: "GREENLOOP_USER", userId: user.id, firstName: user.firstName, lastName: user.lastName });
}
function parseQrPayload(raw) {
  try {
    const obj = JSON.parse(String(raw || ""));
    if (obj?.type === "GREENLOOP_USER" && obj?.userId) return obj;
    return null;
  } catch {
    return null;
  }
}

/* =========================
   TRAP (DEMO)
========================= */
function openTrapDemo() {
  Alert.alert("✅ Trappe ouverte", "Démo : la trappe (vérin électrique) s’ouvre après scan du QR code.");
}


/* =========================
   LIVE SENSOR (SIMU)
========================= */
function clamp01(n) {
  if (!Number.isFinite(n)) return 0;
  return Math.max(0, Math.min(1, n));
}
function clampPct(n) {
  if (!Number.isFinite(n)) return 0;
  return Math.max(0, Math.min(100, n));
}
function approxMove(current, target, speed = 0.12) {
  // rapproche doucement current vers target
  const c = Number(current);
  const t = Number(target);
  if (!Number.isFinite(c) || !Number.isFinite(t)) return t;
  return c + (t - c) * speed;
}
function fanStateFromConfig(cfg, tempC, nowMs) {
  const mode = cfg?.fanMode || "OFF";
  const thr = Number(cfg?.fanThresholdTemp ?? 65);
  const intervalH = Math.max(0.25, Number(cfg?.fanIntervalHours ?? 2));
  if (mode === "OFF") return "OFF";
  if (mode === "ON_REG") return tempC >= thr ? "ON" : "OFF";
  // INTERMITTENT : ON sur une petite fenêtre au début de chaque période
  const periodMs = intervalH * 3600_000;
  const phase = (nowMs % periodMs) / periodMs; // 0..1
  return phase < 0.12 ? "ON" : "OFF";
}

/* =========================
   MAIN APP
========================= */
export default function App() {
  // Theme
  const [themeMode, setThemeMode] = useState("dark");
  const THEME = useMemo(() => (themeMode === "light" ? LIGHT : DARK), [themeMode]);
  const styles = useMemo(() => makeStyles(THEME), [THEME]);
  const placeholderColor = THEME.mode === "dark" ? "rgba(255,255,255,0.25)" : "rgba(0,0,0,0.25)";

  // Screens
  const [screen, setScreen] = useState("MAIN"); // MAIN | IMPACT | BOUTIQUE | LIVE | IDENTITYQR | AUTH
  const [liveSubTab, setLiveSubTab] = useState("CUVES"); // CUVES | CUVE4

  // Drawer
  const [drawerOpen, setDrawerOpen] = useState(false);
  const drawerX = useRef(new Animated.Value(-SCREEN_W)).current;
  const overlayA = useRef(new Animated.Value(0)).current;

  function openDrawer() {
    setDrawerOpen(true);
    Animated.parallel([
      Animated.timing(drawerX, {
        toValue: 0,
        duration: 220,
        useNativeDriver: true,
        easing: Easing.out(Easing.cubic),
      }),
      Animated.timing(overlayA, {
        toValue: 1,
        duration: 220,
        useNativeDriver: true,
      }),
    ]).start();
  }

  function closeDrawer() {
    Animated.parallel([
      Animated.timing(drawerX, {
        toValue: -SCREEN_W,
        duration: 200,
        useNativeDriver: true,
        easing: Easing.in(Easing.cubic),
      }),
      Animated.timing(overlayA, {
        toValue: 0,
        duration: 200,
        useNativeDriver: true,
      }),
    ]).start(() => setDrawerOpen(false));
  }

  const [drawerPage, setDrawerPage] = useState("HOME"); // HOME | IDENTIFY | SETTINGS | ADMIN

  // Session admin
  const [session, setSession] = useState(null);
  const [adminPin, setAdminPin] = useState(DEFAULT_ADMIN_PIN);
  const [pinInput, setPinInput] = useState("");

  // Users (local)
  const [users, setUsers] = useState({});
  const [currentUserId, setCurrentUserId] = useState(null);

  // Profil (utilisateur courant)
  const [profile, setProfile] = useState({ firstName: "", lastName: "", zip: "" });
  const [profileValidated, setProfileValidated] = useState(false);

  // === IDENTITÉ + QR (helpers) ===
  function normalizeName(v) {
    return String(v || "").trim().toLowerCase();
  }

  const validateIdentityProfile = useCallback(() => {
    const fRaw = String(profile.firstName || "").trim();
    const lRaw = String(profile.lastName || "").trim();
    const zRaw = String(profile.zip || "").trim();

    if (!fRaw || !lRaw || !zRaw) {
      Alert.alert("Champs requis", "Remplis prénom, nom et code postal puis appuie sur Valider.");
      return;
    }

    const f = normalizeName(fRaw);
    const l = normalizeName(lRaw);

    // Cherche un utilisateur existant (prénom+nom)
    const found = Object.values(users || {}).find(
      (u) => normalizeName(u.firstName) === f && normalizeName(u.lastName) === l
    );

    if (found) {
      // met à jour le code postal si nécessaire
      if (zRaw && (!found.zip || String(found.zip).trim() !== zRaw)) {
        setUsers((prev) => ({ ...prev, [found.id]: { ...(prev[found.id] || {}), zip: zRaw } }));
      }
      setCurrentUserId(found.id);
      setDeposits(Array.isArray(found.deposits) ? found.deposits : []);
      setSpentPoints(Number(found.spentPoints) || 0);
      setProfile({ firstName: String(found.firstName || ""), lastName: String(found.lastName || ""), zip: String(zRaw || found.zip || "") });
      setProfileValidated(true);
      return;
    }

    // Crée un nouveau compte uniquement au clic "Valider"
    const id = makeUserId(fRaw, lRaw);
    const nu = { id, firstName: fRaw, lastName: lRaw, zip: zRaw, deposits: [], spentPoints: 0 };
    setUsers((prev) => ({ ...prev, [id]: nu }));
    setCurrentUserId(id);
    setDeposits([]);
    setSpentPoints(0);
    setProfileValidated(true);
  }, [profile.firstName, profile.lastName, profile.zip, users]);


  const deleteUser = useCallback(
    (userId) => {
      Alert.alert("Supprimer", "Supprimer ce compte utilisateur ? (points & historique)", [
        { text: "Annuler", style: "cancel" },
        {
          text: "Supprimer",
          style: "destructive",
          onPress: () => {
            setUsers((prev) => {
              const next = { ...(prev || {}) };
              delete next[userId];
              return next;
            });

            if (userId === currentUserId) {
              const remaining = Object.keys(users || {}).filter((id) => id !== userId);
              const nextId = remaining.length ? remaining[0] : null;
              setCurrentUserId(nextId);

              if (nextId && users?.[nextId]) {
                const u = users[nextId];
                setProfile({ firstName: u.firstName || "", lastName: u.lastName || "", zip: u.zip || "" });
                        setProfileValidated(true);
                setProfileValidated(true);
                setDeposits(Array.isArray(u.deposits) ? u.deposits : []);
                setSpentPoints(Number(u.spentPoints) || 0);
              } else {
                setProfile({ firstName: "", lastName: "", zip: "" });
                setDeposits([]);
                setSpentPoints(0);
                setProfileValidated(false);
              }
            }
          },
        },
      ]);
    },
    [currentUserId, users]
  );


  // Data + location
  const [userCoords, setUserCoords] = useState(DEFAULT_USER_COORDS);
  const [hasLocation, setHasLocation] = useState(false);
  const [composters, setComposters] = useState([]);
  const [deposits, setDeposits] = useState([]);

  // Boutique
  const [spentPoints, setSpentPoints] = useState(0);

  // Pilotage à distance
  const [remoteConfig, setRemoteConfig] = useState(DEFAULT_REMOTE_CONFIG);
  const [remoteTab, setRemoteTab] = useState("TEMP"); // TEMP | HUMID | FAN | ACTION
  const [remoteModeKey, setRemoteModeKey] = useState("REMPLISSAGE");
  const [lastPush, setLastPush] = useState(null);

  // Tabs user
  const [activeTab, setActiveTab] = useState("ACCUEIL"); // ACCUEIL | CARTE | DEPOTS | CLASSEMENT

  // ✅ Admin tabs (CUVES supprimé)
  const [adminTab, setAdminTab] = useState("DASH"); // DASH | PILOTAGE | COMPOSTEURS

  // Form dépôt
  const [selectedComposterId, setSelectedComposterId] = useState(null);
  const [wasteKg, setWasteKg] = useState("");
  const [note, setNote] = useState("");

  // Admin create composter fields
  const [aName, setAName] = useState("");
  const [aAddress, setAAddress] = useState("");
  const [aLat, setALat] = useState("");
  const [aLon, setALon] = useState("");

  // Boot
  const [booting, setBooting] = useState(true);
  const bootP = useRef(new Animated.Value(0)).current;

  // ✅ capteurs temps réel (SIMU)
  // structure: { [composterId]: { [vatId]: { tempC, humidity, airQuality, gasPpm, fan, action, fillPct?, updatedAt } } }
  const [liveSensors, setLiveSensors] = useState({});

  useEffect(() => {
    Animated.timing(bootP, { toValue: 1, duration: 3000, useNativeDriver: false, easing: Easing.linear }).start(() => setBooting(false));
  }, [bootP]);

  // Load data
  useEffect(() => {
    (async () => {
      const [
        savedTheme,
        savedProfile,
        savedUsers,
        savedCurrentUser,
        savedPin,
        savedSession,
        savedComposters,
        savedDeposits,
        savedSpentPoints,
        savedRemoteConfig,
        savedLastPush,
      ] = await Promise.all([
        readJSON(K_THEME, "dark"),
        readJSON(K_PROFILE, { firstName: "", lastName: "", zip: "" }),
        readJSON(K_USERS, null),
        readJSON(K_CURRENT_USER, null),
        readJSON(K_ADMIN_PIN, DEFAULT_ADMIN_PIN),
        readJSON(K_SESSION, null),
        readJSON(K_COMPOSTERS, null),
        readJSON(K_DEPOSITS, []),
        readJSON(K_SPENT_POINTS, 0),
        readJSON(K_REMOTE_CONFIG, DEFAULT_REMOTE_CONFIG),
        readJSON(K_LAST_PUSH, null),
      ]);

      setThemeMode(savedTheme === "light" ? "light" : "dark");

      // Users (migration simple)
      let uMap = (savedUsers && typeof savedUsers === "object") ? savedUsers : null;
      if (!uMap || !Object.keys(uMap).length) {
        const baseProfile = savedProfile || { firstName: "", lastName: "", zip: "" };
        const id = makeUserId(baseProfile.firstName, baseProfile.lastName);
        uMap = {
          [id]: {
            id,
            firstName: baseProfile.firstName || "",
            lastName: baseProfile.lastName || "",
            zip: baseProfile.zip || "",
            deposits: [],
            spentPoints: 0,
          },
        };
      }
      const firstId = savedCurrentUser && uMap[savedCurrentUser] ? savedCurrentUser : Object.keys(uMap)[0];
      const activeUser = uMap[firstId];
      setUsers(uMap);
      setCurrentUserId(firstId);
      setProfile({ firstName: activeUser?.firstName || "", lastName: activeUser?.lastName || "", zip: activeUser?.zip || "" });

      setDeposits(Array.isArray(activeUser?.deposits) ? activeUser.deposits : []);
      setSpentPoints(Number(activeUser?.spentPoints) || 0);
      setAdminPin(String(savedPin || DEFAULT_ADMIN_PIN));
      setSession(savedSession);
      setRemoteConfig(savedRemoteConfig || DEFAULT_REMOTE_CONFIG);
      setLastPush(savedLastPush);

      const initialComposters =
        Array.isArray(savedComposters) && savedComposters.length ? savedComposters : SEED_COMPOSTERS;

      const compostersWithVats = initialComposters.map((c) => {
        const vats = Array.isArray(c.vats) ? c.vats.slice(0, 4) : [];
        return {
          ...c,
          vats: [
            vats[0] || { id: "v1", label: "Cuve 1", tempC: 40, mode: "REMPLISSAGE" },
            vats[1] || { id: "v2", label: "Cuve 2", tempC: 55, mode: "MATURATION" },
            vats[2] || { id: "v3", label: "Cuve 3", tempC: 30, mode: "PRET" },
            vats[3] || { id: "v4", label: "Cuve 4 (récup.)", tempC: 26, mode: "PRET" },
          ].map((v) => ({ ...v, tempC: Number.isFinite(Number(v.tempC)) ? Number(v.tempC) : 0 })),
        };
      });

      setComposters(compostersWithVats);
      setSelectedComposterId(compostersWithVats?.[0]?.id ?? null);

      try {
        const { status } = await Location.requestForegroundPermissionsAsync();
        if (status === "granted") {
          const loc = await Location.getCurrentPositionAsync({ accuracy: Location.Accuracy.Balanced });
          setUserCoords({ latitude: loc.coords.latitude, longitude: loc.coords.longitude });
          setHasLocation(true);
        } else {
          setHasLocation(false);
        }
      } catch {
        setHasLocation(false);
      }
    })();
  }, []);

  // Persist
  useEffect(() => { writeJSON(K_THEME, themeMode).catch(() => {}); }, [themeMode]);
  useEffect(() => { writeJSON(K_PROFILE, profile).catch(() => {}); }, [profile]);
  useEffect(() => { writeJSON(K_USERS, users).catch(() => {}); }, [users]);
  useEffect(() => { writeJSON(K_CURRENT_USER, currentUserId).catch(() => {}); }, [currentUserId]);
  useEffect(() => { writeJSON(K_ADMIN_PIN, adminPin).catch(() => {}); }, [adminPin]);
  useEffect(() => { writeJSON(K_SESSION, session).catch(() => {}); }, [session]);
  useEffect(() => { writeJSON(K_COMPOSTERS, composters).catch(() => {}); }, [composters]);
  useEffect(() => { writeJSON(K_DEPOSITS, deposits).catch(() => {}); }, [deposits]);
  useEffect(() => {
    if (!currentUserId) return;
    setUsers((prev) => ({
      ...prev,
      [currentUserId]: { ...(prev[currentUserId] || {}), deposits: Array.isArray(deposits) ? deposits : [] },
    }));
  }, [deposits, currentUserId]);
  useEffect(() => { writeJSON(K_SPENT_POINTS, spentPoints).catch(() => {}); }, [spentPoints]);
  useEffect(() => {
    if (!currentUserId) return;
    setUsers((prev) => ({
      ...prev,
      [currentUserId]: { ...(prev[currentUserId] || {}), spentPoints: Number(spentPoints) || 0 },
    }));
  }, [spentPoints, currentUserId]);
  useEffect(() => { writeJSON(K_REMOTE_CONFIG, remoteConfig).catch(() => {}); }, [remoteConfig]);
  useEffect(() => { writeJSON(K_LAST_PUSH, lastPush).catch(() => {}); }, [lastPush]);

  const selectedComposter = useMemo(
    () => composters.find((c) => c.id === selectedComposterId) ?? null,
    [composters, selectedComposterId]
  );

  const nearest = useMemo(() => {
    if (!composters.length) return null;
    let best = null;
    for (const c of composters) {
      const d = kmBetween(userCoords, { latitude: c.latitude, longitude: c.longitude });
      if (!best || d < best.distKm) best = { ...c, distKm: d };
    }
    return best;
  }, [composters, userCoords]);

  const earnedPoints = useMemo(() => deposits.reduce((s, d) => s + pointsFromDepositKg(d.wasteKg || 0), 0), [deposits]);
  const totalPoints = useMemo(() => Math.max(0, earnedPoints - (Number(spentPoints) || 0)), [earnedPoints, spentPoints]);

  const leaderboard = useMemo(() => {
    const list = Object.values(users || {}).map((u) => {
      const earned = (Array.isArray(u?.deposits) ? u.deposits : []).reduce(
        (s, d) => s + pointsFromDepositKg(d?.wasteKg || 0),
        0
      );
      const spent = Number(u?.spentPoints) || 0;
      const points = Math.max(0, earned - spent);
      const name = `${u?.firstName || ""} ${u?.lastName || ""}`.trim() || "Utilisateur";
      return { id: u.id, name, points };
    });

    // Petits "bots" optionnels pour remplir le classement (démo)
    const base = list.length ? Math.max(...list.map((x) => x.points)) : 0;
    const bots = [
      { id: "b1", name: "Alex", points: Math.max(0, Math.round(base * 0.9) + 30) },
      { id: "b2", name: "Maya", points: Math.max(0, Math.round(base * 1.15) + 10) },
      { id: "b3", name: "Nolan", points: Math.max(0, Math.round(base * 0.75) + 55) },
    ];

    return [...list, ...bots]
      .sort((a, b) => b.points - a.points)
      .map((x, i) => ({ ...x, rank: i + 1 }));
  }, [users]);

  const isAdmin = session?.role === "ADMIN";

  const impact = useMemo(() => {
    const totalWasteKg = deposits.reduce((s, d) => s + (Number(d.wasteKg) || 0), 0);
    const totalCompostKg = deposits.reduce((s, d) => s + (Number(d.compostKg) || 0), 0);
    const co2AvoidedKg = totalWasteKg * 0.25;
    const collectionsAvoided = Math.floor(totalWasteKg / 20);
    const biodiversityPct = Math.max(10, Math.min(99, Math.round(35 + (totalWasteKg / 60) * 60)));
    return { totalWasteKg, totalCompostKg, co2AvoidedKg, collectionsAvoided, biodiversityPct };
  }, [deposits]);

  // ✅ LIVE SENSOR SIMU (toutes les 2s)
  useEffect(() => {
    if (!composters.length) return;

    // init si vide
    setLiveSensors((prev) => {
      const next = { ...prev };
      for (const c of composters) {
        if (!next[c.id]) next[c.id] = {};
        for (const v of (c.vats || []).slice(0, 4)) {
          if (!next[c.id][v.id]) {
            const cfg = remoteConfig[v.mode] || DEFAULT_REMOTE_CONFIG[v.mode];
            const hum = Number(cfg?.humidity ?? 50);
            next[c.id][v.id] = {
              tempC: Number(v.tempC ?? 0),
              humidity: hum,
              airQuality: 75,
              gasPpm: 320,
              fillPct: v.id === "v4" ? 12 : undefined,
              fan: "OFF",
              action: String(cfg?.physicalAction ?? ""),
              updatedAt: new Date().toISOString(),
            };
          }
        }
      }
      return next;
    });

    const id = setInterval(() => {
      const now = Date.now();

      setLiveSensors((prev) => {
        const next = { ...prev };

        for (const c of composters) {
          if (!next[c.id]) next[c.id] = {};
          for (const v of (c.vats || []).slice(0, 4)) {
            const cfg = remoteConfig[v.mode] || DEFAULT_REMOTE_CONFIG[v.mode];
            const tMin = Number(cfg?.tempMin ?? 20);
            const tMax = Number(cfg?.tempMax ?? 30);
            const targetT = (tMin + tMax) / 2;

            const targetH = Number(cfg?.humidity ?? 50);

            const cur = next[c.id][v.id] || {
              tempC: Number(v.tempC ?? 0),
              humidity: targetH,
              airQuality: 75,
              gasPpm: 320,
              fillPct: v.id === "v4" ? 12 : undefined,
              fan: "OFF",
              action: String(cfg?.physicalAction ?? ""),
              updatedAt: new Date().toISOString(),
            };

            // bruit léger
            const noiseT = (Math.random() - 0.5) * 0.6;
            const noiseH = (Math.random() - 0.5) * 1.6;

            const tempNext = clampTemp(approxMove(cur.tempC, targetT, 0.18) + noiseT);
            const humNext = clampPct(approxMove(cur.humidity, targetH, 0.14) + noiseH);

            // Qualité d’air (0..100) + Gaz (ppm) - démo
            const aqTarget = v.mode === "REMPLISSAGE" ? 72 : v.mode === "MATURATION" ? 64 : 80;
            const gasTarget = v.mode === "REMPLISSAGE" ? 420 : v.mode === "MATURATION" ? 680 : 260;
            const aqNoise = (Math.random() - 0.5) * 2.2;
            const gasNoise = (Math.random() - 0.5) * 22;
            const airQuality = clampPct(approxMove(cur.airQuality ?? aqTarget, aqTarget, 0.10) + aqNoise);
            const gasPpm = Math.max(0, approxMove(cur.gasPpm ?? gasTarget, gasTarget, 0.12) + gasNoise);

            // Cuve 4 : taux de remplissage (0..100) - démo
            let fillPct = cur.fillPct;
            if (v.id === "v4") {
              const totalWasteForComp = deposits
                .filter((d) => d.composterId === c.id)
                .reduce((s, d) => s + (Number(d.wasteKg) || 0), 0);
              const targetFill = clampPct((totalWasteForComp * 6) % 100);
              fillPct = clampPct(approxMove(Number(fillPct ?? 0), targetFill, 0.08) + (Math.random() - 0.5) * 1.2);
            }

            const fan = fanStateFromConfig(cfg, tempNext, now);
            const action = String(cfg?.physicalAction ?? "");

            next[c.id][v.id] = {
              tempC: tempNext,
              humidity: humNext,
              airQuality,
              gasPpm,
              fillPct,
              fan,
              action,
              updatedAt: new Date().toISOString(),
            };
          }
        }
        return next;
      });
    }, 2000);

    return () => clearInterval(id);
  }, [composters, remoteConfig, deposits]);
function signOutAdmin() {
    setSession(null);
    setAdminTab("DASH");
    setActiveTab("ACCUEIL");
    closeDrawer();
  }

  function enterAdminFromDrawer() {
    const pin = String(pinInput || "").trim();
    if (!pin) return Alert.alert("PIN requis", "Entre le code admin.");
    if (pin !== String(adminPin)) return Alert.alert("Erreur", "PIN incorrect.");
    setSession(ADMIN);
    setPinInput("");
    closeDrawer();
  }

  function onCreateDeposit() {
    if (!selectedComposter) {
      Alert.alert("Erreur", "Sélectionne un composteur.");
      return;
    }
    const kg = Number(String(wasteKg).replace(",", "."));
    if (!Number.isFinite(kg) || kg <= 0) {
      Alert.alert("Entrée invalide", "Indique un poids en kg (ex: 2.5).");
      return;
    }
    const compostKg = Math.round(kg * COMPOST_RATIO * 100) / 100;
    const now = new Date();
    const maturesAt = addDays(now, MATURATION_DAYS);
    const d = {
      id: uid("dep"),
      createdAt: now.toISOString(),
      maturesAt: maturesAt.toISOString(),
      wasteKg: kg,
      compostKg,
      note: note.trim(),
      composterId: selectedComposter.id,
      composterName: selectedComposter.name,
    };
    setDeposits((prev) => [d, ...prev]);
    setWasteKg("");
    setNote("");
    setActiveTab("DEPOTS");
  }

  function onDeleteDeposit(id) {
    Alert.alert("Supprimer", "Supprimer ce dépôt ? (les points baisseront)", [
      { text: "Annuler", style: "cancel" },
      { text: "Supprimer", style: "destructive", onPress: () => setDeposits((p) => p.filter((x) => x.id !== id)) },
    ]);
  }

  function onDeleteUser(userId) {
    const uidToDelete = String(userId || "");
    if (!uidToDelete) return;

    const isCurrent = uidToDelete === currentUserId;
    const usersCount = Object.keys(users || {}).length;

    Alert.alert(
      "Supprimer l’utilisateur",
      usersCount <= 1
        ? "Tu ne peux pas supprimer le dernier compte."
        : "Supprimer ce compte ? Les points et l’historique associés seront supprimés.",
      [
        { text: "Annuler", style: "cancel" },
        usersCount <= 1
          ? { text: "OK", style: "default" }
          : {
              text: "Supprimer",
              style: "destructive",
              onPress: () => {
                setUsers((prev) => {
                  const next = { ...(prev || {}) };
                  delete next[uidToDelete];
                  return next;
                });

                if (isCurrent) {
                  const remainingIds = Object.keys(users || {}).filter((id) => id !== uidToDelete);
                  const nextId = remainingIds[0] || null;
                  setCurrentUserId(nextId);

                  const nextUser = nextId ? (users || {})[nextId] : null;
                  setProfile({
                    firstName: nextUser?.firstName || "",
                    lastName: nextUser?.lastName || "",
                    zip: nextUser?.zip || "",
                  });
                  setDeposits(Array.isArray(nextUser?.deposits) ? nextUser.deposits : []);
                  setSpentPoints(Number(nextUser?.spentPoints) || 0);
                }
              },
            },
      ]
    );
  }


  function onAddComposter() {
    if (!isAdmin) {
      Alert.alert("Accès refusé", "Mode admin requis.");
      return;
    }
    const lat = Number(String(aLat).replace(",", "."));
    const lon = Number(String(aLon).replace(",", "."));
    if (!aName.trim() || !aAddress.trim() || !Number.isFinite(lat) || !Number.isFinite(lon)) {
      Alert.alert("Champs requis", "Nom, adresse, latitude et longitude sont obligatoires.");
      return;
    }
    const c = {
      id: uid("comp"),
      name: aName.trim(),
      address: aAddress.trim(),
      latitude: lat,
      longitude: lon,
      accepts: ["Épluchures", "Marc de café"],
      vats: [
        { id: "v1", label: "Cuve 1", tempC: 40, mode: "REMPLISSAGE" },
        { id: "v2", label: "Cuve 2", tempC: 55, mode: "MATURATION" },
        { id: "v3", label: "Cuve 3", tempC: 30, mode: "PRET" },
        { id: "v4", label: "Cuve 4 (récup.)", tempC: 26, mode: "PRET" },
      ],
    };
    setComposters((prev) => [c, ...prev]);
    setAName("");
    setAAddress("");
    setALat("");
    setALon("");
    setSelectedComposterId(c.id);
    Alert.alert("✅ OK", "Composteur ajouté.");
  }

  function onDeleteComposter(composterId) {
    Alert.alert("Supprimer", "Supprimer ce composteur ? (irréversible)", [
      { text: "Annuler", style: "cancel" },
      {
        text: "Supprimer",
        style: "destructive",
        onPress: () => {
          setComposters((prev) => prev.filter((c) => c.id !== composterId));
          if (selectedComposterId === composterId) {
            const next = composters.find((c) => c.id !== composterId)?.id ?? null;
            setSelectedComposterId(next);
          }
        },
      },
    ]);
  }

  function redeemOffer(offer) {
    if (totalPoints < offer.cost) {
      Alert.alert("Points insuffisants", `Il te faut ${offer.cost} points pour ${offer.kg} kg.`);
      return;
    }
    Alert.alert("Confirmer", `Retirer ${offer.kg} kg de compost contre ${offer.cost} points ?`, [
      { text: "Annuler", style: "cancel" },
      {
        text: "Confirmer",
        onPress: () => {
          setSpentPoints((p) => (Number(p) || 0) + offer.cost);
          Alert.alert("✅ Commande validée", `Tu as commandé ${offer.kg} kg de compost.`);
        },
      },
    ]);
  }

  function patchRemote(modeKey, patch) {
    setRemoteConfig((prev) => ({ ...prev, [modeKey]: { ...(prev[modeKey] || {}), ...patch } }));
  }

  function sendRemoteConfigToComposter(composterId) {
    const comp = composters.find((c) => c.id === composterId);
    if (!comp) return;

    const payload = {
      composterId,
      composterName: comp.name,
      sentAt: new Date().toISOString(),
      config: remoteConfig,
    };

    setLastPush(payload);

    Alert.alert(
      "📡 Envoyé",
      `Paramètres envoyés à "${comp.name}".\n\n(Mode démo : ici tu brancheras l’API/ESP32 plus tard.)`
    );
  }

  /* =========================
     BOOT
  ========================= */
  if (booting) {
    const w = bootP.interpolate({ inputRange: [0, 1], outputRange: ["0%", "100%"] });
    return (
      <View style={[styles.boot, { backgroundColor: THEME.bg }]}>
        <StatusBar barStyle={THEME.mode === "dark" ? "light-content" : "dark-content"} />
        <Image source={LOGO} style={styles.bootLogo} resizeMode="contain" />
        <Text style={styles.bootTitle}>GreenLoop</Text>
        <Text style={styles.bootSub}>Démarrage…</Text>

        <View style={styles.bootBarOuter}>
          <Animated.View style={[styles.bootBarInner, { width: w }]} />
        </View>

        <Text style={styles.bootHint}>Ouverture automatique (3s)</Text>
      </View>
    );
  }

  
  /* =========================
     IDENTIFY + QR (NO SCAN)
  ========================= */
  if (screen === "IDENTITYQR") {
    const activeUser = currentUserId ? users?.[currentUserId] : null;
    const qrValue = activeUser ? buildQrPayload(activeUser) : "";

    return (
      <SafeAreaView style={styles.app}>
        <StatusBar barStyle={THEME.mode === "dark" ? "light-content" : "dark-content"} />

        <View style={styles.liveHeader}>
          <Pressable style={styles.impactBackBtn} onPress={() => setScreen("MAIN")}>
            <Text style={styles.impactBackTxt}>← Retour</Text>
          </Pressable>

          <View style={{ flex: 1 }}>
            <Text style={styles.liveTitle}>🧾 S’identifier</Text>
            <Text style={styles.liveSub}>Remplis ton prénom & nom : ton QR code apparaît automatiquement.</Text>
          </View>
        </View>

        <ScrollView contentContainerStyle={{ padding: 16, paddingBottom: 26 }}>
          <View style={styles.card}>
            <Text style={styles.cardTitle}>Infos utilisateur</Text>

            <Text style={styles.label}>Prénom</Text>
            <TextInput
              value={profile.firstName}
              onChangeText={(t) => {
                setProfile((p) => ({ ...p, firstName: t }));
                setProfileValidated(false);
              }}
              placeholder="Ex: Sara"
              placeholderTextColor={placeholderColor}
              style={styles.input}
            />

            <Text style={styles.label}>Nom</Text>
            <TextInput
              value={profile.lastName}
              onChangeText={(t) => {
                setProfile((p) => ({ ...p, lastName: t }));
                setProfileValidated(false);
              }}
              placeholder="Ex: Martin"
              placeholderTextColor={placeholderColor}
              style={styles.input}
            />

            <Text style={styles.label}>Code postal</Text>
            <TextInput
              value={profile.zip}
              onChangeText={(t) => {
                setProfile((p) => ({ ...p, zip: t }));
                setProfileValidated(false);
              }}
              placeholder="Ex: 92500"
              placeholderTextColor={placeholderColor}
              style={styles.input}
              keyboardType={Platform.select({ ios: "number-pad", android: "numeric" })}
            />

            <Pressable
              style={[styles.primaryBtn, { marginTop: 12 }]}
              onPress={validateIdentityProfile}
            >
              <Text style={styles.primaryBtnText}>Valider</Text>
            </Pressable>
          </View>

          <View style={styles.card}>
            <Text style={styles.cardTitle}>QR code</Text>

            {profileValidated && activeUser && String(activeUser.firstName || "").trim() && String(activeUser.lastName || "").trim() ? (
              <View style={{ alignItems: "center" }}>
                <QRCode value={qrValue} size={230} />
                <Text style={[styles.cardMuted, { marginTop: 12, textAlign: "center" }]}>
                  QR pour {activeUser.firstName} {activeUser.lastName}
                </Text>

                <Pressable
                  style={[styles.secondaryBtn, { marginTop: 12, borderColor: "rgba(255,42,42,0.35)" }]}
                  onPress={() => deleteUser(activeUser.id)}
                >
                  <Text style={[styles.secondaryBtnText, { color: THEME.red }]}>🗑️ Supprimer ce compte</Text>
                </Pressable>
              </View>
            ) : (
              <Text style={styles.cardMuted}>Remplis prénom, nom et code postal puis appuie sur Valider pour générer ton QR code.</Text>
            )}
          </View>

          <View style={styles.card}>
            <Text style={styles.cardTitle}>Utilisateurs enregistrés</Text>

            {Object.values(users || {}).length ? (
              Object.values(users || {}).map((u) => {
                const active = u.id === currentUserId;
                return (
                  <View key={u.id} style={[styles.rowItem, { alignItems: "flex-start" }]}>
                    <Pressable
                      style={{ flex: 1 }}
                      onPress={() => {
                        setCurrentUserId(u.id);
                        setProfile({ firstName: u.firstName || "", lastName: u.lastName || "", zip: u.zip || "" });
                        setDeposits(Array.isArray(u.deposits) ? u.deposits : []);
                        setSpentPoints(Number(u.spentPoints) || 0);
                      }}
                    >
                      <Text style={styles.rowTitle}>{active ? "✅ " : ""}{u.firstName} {u.lastName}</Text>
                      <Text style={styles.rowSub}>ID: {u.id}</Text>
                    </Pressable>

                    <Pressable
                      onPress={() => deleteUser(u.id)}
                      style={[styles.secondaryBtnSmall, { borderColor: "rgba(255,42,42,0.35)" }]}
                    >
                      <Text style={[styles.secondaryBtnText, { color: THEME.red }]}>🗑️</Text>
                    </Pressable>
                  </View>
                );
              })
            ) : (
              <Text style={styles.cardMuted}>Aucun utilisateur.</Text>
            )}
          </View>
        </ScrollView>
      </SafeAreaView>
    );
  }

/* =========================
     LIVE USER PAGE (READ ONLY)
  ========================= */
  if (screen === "LIVE") {
    const comp = selectedComposter;

    return (
      <SafeAreaView style={styles.app}>
        <StatusBar barStyle={THEME.mode === "dark" ? "light-content" : "dark-content"} />

        <View style={styles.liveHeader}>
          <Pressable style={styles.impactBackBtn} onPress={() => setScreen("MAIN")}>
            <Text style={styles.impactBackTxt}>← Retour</Text>
          </Pressable>

          <View style={{ flex: 1 }}>
            <Text style={styles.liveTitle}>📟 Suivi en temps réel</Text>
            <Text style={styles.liveSub}>Données capteurs (lecture seule)</Text>
          </View>
        </View>

        <ScrollView contentContainerStyle={{ paddingBottom: 24 }}>
          <View style={styles.card}>
            <Text style={styles.cardTitle}>Composteur</Text>
            <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
              {composters.map((c) => {
                const active = c.id === selectedComposterId;
                return (
                  <Pressable
                    key={c.id}
                    onPress={() => setSelectedComposterId(c.id)}
                    style={[styles.chip, active && styles.chipActive]}
                  >
                    <Text style={[styles.chipText, active && styles.chipTextActive]} numberOfLines={1}>
                      📍 {c.name}
                    </Text>
                  </Pressable>
                );
              })}
            </ScrollView>

            {comp ? (
              <Text style={[styles.cardMuted, { marginTop: 10 }]}>
                🕒 24/24 • 7j/7 • 🔓 Accès libre{"\n"}
                Adresse : {comp.address}
              </Text>
            ) : (
              <Text style={styles.cardMuted}>Aucun composteur.</Text>
            )}
          </View>

          {comp ? (
            <View style={styles.card}>
              <Text style={styles.cardTitle}>Cuves (capteurs)</Text>

              {/* sous-onglets */}
              <View style={styles.liveTabsRow}>
                <Pressable style={[styles.liveTabBtn, liveSubTab === "CUVES" && styles.liveTabBtnActive]} onPress={() => setLiveSubTab("CUVES")}>
                  <Text style={[styles.liveTabBtnText, liveSubTab === "CUVES" && styles.liveTabBtnTextActive]}>Cuves 1–3</Text>
                </Pressable>
                <Pressable style={[styles.liveTabBtn, liveSubTab === "CUVE4" && styles.liveTabBtnActive]} onPress={() => setLiveSubTab("CUVE4")}>
                  <Text style={[styles.liveTabBtnText, liveSubTab === "CUVE4" && styles.liveTabBtnTextActive]}>Cuve 4 • Remplissage</Text>
                </Pressable>
              </View>

              {liveSubTab === "CUVES" ? (
                (comp.vats || []).slice(0, 3).map((v) => {
                const s = liveSensors?.[comp.id]?.[v.id];
                const modeLabel = VAT_MODES.find((m) => m.key === v.mode)?.label ?? v.mode;
                const updated = s?.updatedAt ? formatDateFR(s.updatedAt) : "—";

                return (
                  <View key={v.id} style={styles.liveVatCard}>
                    <View style={{ flexDirection: "row", justifyContent: "space-between", alignItems: "center" }}>
                      <Text style={styles.liveVatTitle}>🧪 {v.label}</Text>
                      <View style={styles.liveModePill}>
                        <Text style={styles.liveModePillText}>{modeLabel}</Text>
                      </View>
                    </View>

                    <View style={styles.liveRow}>
                      <View style={styles.liveKpi}>
                        <Text style={styles.liveKpiLabel}>🌡️ Température</Text>
                        <Text style={styles.liveKpiValue}>{Number(s?.tempC ?? v.tempC ?? 0).toFixed(1)}°C</Text>
                      </View>
                      <View style={styles.liveKpi}>
                        <Text style={styles.liveKpiLabel}>💧 Humidité</Text>
                        <Text style={styles.liveKpiValue}>{Number(s?.humidity ?? 0).toFixed(0)}%</Text>
                      </View>
                    </View>

                    <View style={styles.liveRow}>
                      <View style={styles.liveKpi}>
                        <Text style={styles.liveKpiLabel}>🌬️ Qualité d’air</Text>
                        <Text style={styles.liveKpiValue}>{Number(s?.airQuality ?? 0).toFixed(0)}%</Text>
                      </View>
                      <View style={styles.liveKpi}>
                        <Text style={styles.liveKpiLabel}>🧪 Gaz</Text>
                        <Text style={styles.liveKpiValue}>{Number(s?.gasPpm ?? 0).toFixed(0)} ppm</Text>
                      </View>
                    </View>

                    <View style={styles.liveRow}>
                      <View style={styles.liveKpiWide}>
                        <Text style={styles.liveKpiLabel}>🌀 Ventilateur</Text>
                        <Text style={styles.liveKpiValue}>{String(s?.fan ?? "OFF")}</Text>
                      </View>
                    </View>

                    <View style={styles.liveActionBox}>
                      <Text style={styles.liveActionTitle}>🔁 Mélange du compost</Text>
                      <Text style={styles.liveActionText}>{String(s?.action ?? "").trim() || "—"}</Text>
                    </View>

                    <Text style={[styles.cardMuted, { marginTop: 10 }]}>
                      Dernière mise à jour : {updated}
                    </Text>
                  </View>
                );
                })
              ) : (
                <View style={styles.liveVatCard}>
                  {(() => {
                    const v4 = (comp.vats || []).find((x) => x.id === "v4");
                    const s4 = v4 ? liveSensors?.[comp.id]?.[v4.id] : null;
                    const pct = Number(s4?.fillPct ?? 0);
                    const updated4 = s4?.updatedAt ? formatDateFR(s4.updatedAt) : "—";
                    return (
                      <>
                        <View style={{ flexDirection: "row", justifyContent: "space-between", alignItems: "center" }}>
                          <Text style={styles.liveVatTitle}>🧺 {v4?.label || "Cuve 4 (récup.)"}</Text>
                          <View style={styles.liveModePill}>
                            <Text style={styles.liveModePillText}>Récupération</Text>
                          </View>
                        </View>

                        <View style={styles.cuve4Big}>
                          <Text style={styles.cuve4BigValue}>{pct.toFixed(0)}%</Text>
                          <Text style={styles.cuve4BigLabel}>Taux de remplissage</Text>
                        </View>

                        <View style={styles.cuve4BarOuter}>
                          <View style={[styles.cuve4BarInner, { width: `${clampPct(pct)}%` }]} />
                        </View>

                        <View style={styles.liveRow}>
                          <View style={styles.liveKpi}>
                            <Text style={styles.liveKpiLabel}>🌬️ Qualité d’air</Text>
                            <Text style={styles.liveKpiValue}>{Number(s4?.airQuality ?? 0).toFixed(0)}%</Text>
                          </View>
                          <View style={styles.liveKpi}>
                            <Text style={styles.liveKpiLabel}>🧪 Gaz</Text>
                            <Text style={styles.liveKpiValue}>{Number(s4?.gasPpm ?? 0).toFixed(0)} ppm</Text>
                          </View>
                        </View>

                        <Text style={[styles.cardMuted, { marginTop: 10 }]}>Dernière mise à jour : {updated4}</Text>
                      </>
                    );
                  })()}
                </View>
              )}
            </View>
          ) : null}
        </ScrollView>
      </SafeAreaView>
    );
  }

  /* =========================
     IMPACT
  ========================= */
  if (screen === "IMPACT") {
    return (
      <SafeAreaView style={[styles.app, { backgroundColor: THEME.mode === "dark" ? "#0A1210" : "#F2FBF6" }]}>
        <StatusBar barStyle={THEME.mode === "dark" ? "light-content" : "dark-content"} />

        <View style={styles.impactHeader}>
          <Pressable style={styles.impactBackBtn} onPress={() => setScreen("MAIN")}>
            <Text style={styles.impactBackTxt}>← Retour</Text>
          </Pressable>

          <View style={{ flex: 1 }}>
            <Text style={styles.impactTitle}>Impact écologique</Text>
            <Text style={styles.impactSub}>Tes dépôts font la différence 🌿</Text>
          </View>
        </View>

        <ScrollView contentContainerStyle={{ paddingBottom: 24 }}>
          <View style={styles.impactBigCard}>
            <Text style={styles.impactCardTitle}>Déchets valorisés ♻️</Text>
            <Text style={styles.impactBigNumber}>{impact.totalWasteKg.toFixed(2)} kg</Text>
            <Text style={styles.impactCardDesc}>Total de déchets déposés dans GreenLoop.</Text>
          </View>

          <View style={styles.impactRow}>
            <View style={styles.impactMiniCard}>
              <Text style={styles.impactMiniTitle}>CO₂ évité</Text>
              <Text style={styles.impactMiniValue}>{impact.co2AvoidedKg.toFixed(2)} kg</Text>
            </View>
            <View style={styles.impactMiniCard}>
              <Text style={styles.impactMiniTitle}>Collectes évitées</Text>
              <Text style={styles.impactMiniValue}>{impact.collectionsAvoided}</Text>
            </View>
          </View>

          <View style={styles.impactRow}>
            <View style={styles.impactMiniCard}>
              <Text style={styles.impactMiniTitle}>Compost estimé</Text>
              <Text style={styles.impactMiniValue}>{impact.totalCompostKg.toFixed(2)} kg</Text>
            </View>
            <View style={styles.impactMiniCard}>
              <Text style={styles.impactMiniTitle}>Biodiversité</Text>
              <Text style={styles.impactMiniValue}>{impact.biodiversityPct}%</Text>
            </View>
          </View>
        </ScrollView>
      </SafeAreaView>
    );
  }

  /* =========================
     BOUTIQUE
  ========================= */
  if (screen === "BOUTIQUE") {
    return (
      <SafeAreaView style={styles.app}>
        <StatusBar barStyle="light-content" />

        <View style={styles.shopHeader}>
          <View style={styles.shopHeaderTop}>
            <Text style={styles.shopTitle}>Mes avantages</Text>
            <View style={styles.shopPointsPill}>
              <Text style={styles.shopPointsText}>{totalPoints} Points</Text>
            </View>
          </View>

          <View style={styles.shopTabsRow}>
            <Pressable style={[styles.shopTab, styles.shopTabActive]}>
              <Text style={[styles.shopTabText, styles.shopTabTextActive]}>RÉCOMPENSES</Text>
            </Pressable>
            <Pressable style={styles.shopTab} onPress={() => Alert.alert("Bientôt", "Section bientôt dispo.")}>
              <Text style={styles.shopTabText}>BONS PLANS</Text>
            </Pressable>
</View>
        </View>

        <FlatList
          data={SHOP_OFFERS}
          keyExtractor={(x) => x.id}
          numColumns={2}
          contentContainerStyle={{ padding: 14, paddingBottom: 30 }}
          columnWrapperStyle={{ gap: 12 }}
          ListHeaderComponent={
            <View style={styles.shopTopInfo}>
              <Pressable style={styles.shopBackBtn} onPress={() => setScreen("MAIN")}>
                <Text style={styles.shopBackTxt}>← Retour</Text>
              </Pressable>

              <View style={{ marginTop: 10 }}>
                <Text style={styles.shopInfoText}>✅ 1 kg de déchets = 0.4 kg compost + 100 points</Text>
                <Text style={[styles.shopInfoText, { marginTop: 6 }]}>
                  ⭐ Gagnés : <Text style={styles.shopInfoStrong}>{earnedPoints}</Text> • Dépensés :{" "}
                  <Text style={styles.shopInfoStrong}>{spentPoints}</Text>
                </Text>
              </View>
            </View>
          }
          renderItem={({ item }) => {
            const canBuy = totalPoints >= item.cost;
            return (
              <Pressable onPress={() => redeemOffer(item)} style={[styles.shopCard, !canBuy && { opacity: 0.55 }]}>
                <View style={styles.shopImageWrap}>
                  <Image source={COMPOST_IMG} style={styles.shopImage} resizeMode="contain" />
                </View>

                <Text style={styles.shopCardTitle}>{item.title}</Text>

                <View style={styles.shopCostRow}>
                  <View style={styles.shopCoin} />
                  <Text style={styles.shopCost}>{item.cost}</Text>
                </View>

                <Text style={styles.shopHint}>{canBuy ? "Appuie pour commander" : "Pas assez de points"}</Text>
              </Pressable>
            );
          }}
        />
      </SafeAreaView>
    );
  }

  /* =========================
     MAIN APP
  ========================= */
  return (
    <SafeAreaView style={styles.app}>
      <StatusBar barStyle={THEME.mode === "dark" ? "light-content" : "dark-content"} />

      {/* HEADER */}
      <View style={styles.header}>
        <Pressable
          style={styles.burgerBtn}
          onPress={() => {
            setDrawerPage("HOME");
            openDrawer();
          }}
        >
          <View style={styles.bLine} />
          <View style={styles.bLine} />
          <View style={styles.bLine} />
        </Pressable>

        <View style={styles.headerCenter}>
          <Image source={LOGO} style={styles.headerLogo} resizeMode="contain" />
          <Text style={styles.headerTitleSmall}>
            Bonjour{" "}
            <Text style={{ color: THEME.yellow }}>
              {profile.firstName?.trim() ? profile.firstName.trim() : "!"}
            </Text>
          </Text>
          <Text style={styles.headerSubtitleSmall}>
            {hasLocation ? "📍 Position détectée" : "📍 Position non disponible"} • {composters.length} composteurs
          </Text>
        </View>

        <View style={styles.pointsPill}>
          <Text style={styles.pointsTxt}>{totalPoints}</Text>
          <Text style={styles.pointsSmall}>pts</Text>
        </View>
      </View>

      <View style={styles.content}>
        {/* UTILISATEUR */}
        {!isAdmin && (
          <>
            {activeTab === "ACCUEIL" && (
              <ScrollView contentContainerStyle={{ paddingBottom: 18 }}>
                <View style={styles.hero}>
                  <View style={styles.heroTop}>
                    <Text style={styles.heroKicker}>📊 Tableau de bord</Text>
                    <View style={styles.heroBadge}>
                      <Text style={styles.heroBadgeText}>RATIO {COMPOST_RATIO}</Text>
                    </View>
                  </View>

                  <View style={styles.metricsRow}>
                    <Metric label="⭐ Points" value={String(totalPoints)} styles={styles} />
                    <Metric label="🧾 Dépôts" value={String(deposits.length)} styles={styles} />
                    <Metric
                      label="👑 Rang"
                      value={`#${leaderboard.find((x) => x.id === currentUserId)?.rank ?? "-"}`}
                      styles={styles}
                    />
                  </View>
                </View>

                <View style={styles.card}>
                  <Text style={styles.cardTitle}>📍 Point le plus proche</Text>
                  {nearest ? (
                    <>
                      <Text style={styles.cardMain}>{nearest.name}</Text>
                      <Text style={styles.cardMuted}>
                        {nearest.address} • {nearest.distKm.toFixed(2)} km
                      </Text>

                      <View style={styles.infoStrip}>
                        <Text style={styles.infoStripText}>
                          🕒 Ouvert <Text style={{ fontWeight: "900", color: THEME.text }}>24/24</Text> •{" "}
                          <Text style={{ fontWeight: "900", color: THEME.text }}>7j/7</Text>{"\n"}
                          🔓 Accès libre • Dépôt rapide • Sans réservation
                        </Text>
                      </View>

                      <View style={{ marginTop: 12, flexDirection: "row", gap: 10 }}>
                        <Pressable style={styles.primaryBtn} onPress={() => setActiveTab("CARTE")}>
                          <Text style={styles.primaryBtnText}>🗺️ Carte</Text>
                        </Pressable>
                        <Pressable style={styles.secondaryBtn} onPress={() => setActiveTab("CLASSEMENT")}>
                          <Text style={styles.secondaryBtnText}>👑 Classement</Text>
                        </Pressable>
                      </View>
                    </>
                  ) : (
                    <Text style={styles.cardMuted}>Aucun composteur disponible.</Text>
                  )}
                </View>

                <View style={styles.card}>
                  <Text style={styles.cardTitle}>➕ Nouveau dépôt</Text>

                  <Text style={styles.label}>Composteur</Text>
                  <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
                    {composters.map((c) => {
                      const active = c.id === selectedComposterId;
                      return (
                        <Pressable
                          key={c.id}
                          onPress={() => setSelectedComposterId(c.id)}
                          style={[styles.chip, active && styles.chipActive]}
                        >
                          <Text style={[styles.chipText, active && styles.chipTextActive]} numberOfLines={1}>
                            {c.name}
                          </Text>
                        </Pressable>
                      );
                    })}
                  </ScrollView>

                  <Text style={styles.label}>Poids des déchets (kg)</Text>
                  <TextInput
                    value={wasteKg}
                    onChangeText={setWasteKg}
                    placeholder="Ex: 2.5"
                    placeholderTextColor={placeholderColor}
                    keyboardType={Platform.select({ ios: "decimal-pad", android: "numeric" })}
                    style={styles.input}
                  />

                  <Text style={styles.label}>Note (optionnel)</Text>
                  <TextInput
                    value={note}
                    onChangeText={setNote}
                    placeholder="Ex: épluchures + marc"
                    placeholderTextColor={placeholderColor}
                    style={[styles.input, { height: 46 }]}
                  />

                  <View style={{ marginTop: 12, flexDirection: "row", gap: 10 }}>
                    <Pressable style={styles.primaryBtn} onPress={onCreateDeposit}>
                      <Text style={styles.primaryBtnText}>✅ Enregistrer</Text>
                    </Pressable>
                    <Pressable style={styles.secondaryBtn} onPress={() => setActiveTab("CARTE")}>
                      <Text style={styles.secondaryBtnText}>🗺️ Carte</Text>
                    </Pressable>
                  </View>

                  <View style={styles.infoStrip}>
                    {(() => {
                      const kg = Number(String(wasteKg).replace(",", "."));
                      const compost = Number.isFinite(kg) && kg > 0 ? Math.round(kg * COMPOST_RATIO * 100) / 100 : 0;
                      const pts = pointsFromDepositKg(kg || 0);
                      return (
                        <Text style={styles.infoStripText}>
                          🌱 Compost estimé : <Text style={{ fontWeight: "900", color: THEME.text }}>{compost} kg</Text>
                          {"  "}•{"  "}⭐ +{pts} pts{"  "}•{"  "}Prêt le{" "}
                          <Text style={{ fontWeight: "900", color: THEME.text }}>
                            {formatDateFR(addDays(new Date(), MATURATION_DAYS))}
                          </Text>
                        </Text>
                      );
                    })()}
                  </View>
                </View>
              </ScrollView>
            )}

            {activeTab === "CARTE" && (
              <View style={{ flex: 1 }}>
                <View style={styles.mapTopBar}>
                  <Pressable style={styles.secondaryBtnSmall} onPress={() => setActiveTab("ACCUEIL")}>
                    <Text style={styles.secondaryBtnText}>Retour</Text>
                  </Pressable>
                </View>

                <MapView
                  style={styles.map}
                  initialRegion={{
                    latitude: userCoords.latitude,
                    longitude: userCoords.longitude,
                    latitudeDelta: 0.02,
                    longitudeDelta: 0.02,
                  }}
                  showsUserLocation={hasLocation}
                  userInterfaceStyle={THEME.mode === "dark" ? "dark" : "light"}
                >
                  {composters.map((c) => (
                    <Marker
                      key={c.id}
                      coordinate={{ latitude: c.latitude, longitude: c.longitude }}
                      onPress={() => setSelectedComposterId(c.id)}
                    >
                      <View style={[styles.pin, c.id === selectedComposterId && styles.pinActive]}>
                        <View style={styles.pinDot} />
                      </View>
                      <Callout>
                        <View style={{ maxWidth: 220 }}>
                          <Text style={{ fontWeight: "900" }}>{c.name}</Text>
                          <Text style={{ marginTop: 4, color: "#333" }}>{c.address}</Text>
                          <Text style={{ marginTop: 8, color: "#333" }}>
                            Acceptés : {Array.isArray(c.accepts) ? c.accepts.join(", ") : "—"}
                          </Text>
                          <Text style={{ marginTop: 8, color: "#333" }}>🕒 24/24 • 7j/7 • 🔓 Accès libre</Text>
                        </View>
                      </Callout>
                    </Marker>
                  ))}
                </MapView>

                <View style={styles.mapBottomSheet}>
                  <Text style={styles.sheetTitle}>Sélection</Text>
                  {selectedComposter ? (
                    <>
                      <Text style={styles.sheetMain}>{selectedComposter.name}</Text>
                      <Text style={styles.sheetMuted}>{selectedComposter.address}</Text>
                      <Text style={[styles.sheetMuted, { marginTop: 6 }]}>🕒 24/24 • 7j/7 • 🔓 Accès libre</Text>
                    </>
                  ) : (
                    <Text style={styles.sheetMuted}>Appuie sur un point pour le sélectionner.</Text>
                  )}
                </View>
              </View>
            )}

            {activeTab === "DEPOTS" && (
              <View style={{ flex: 1 }}>
                <View style={styles.listTopBar}>
                  <Text style={styles.sectionTitle}>🧾 Historique des dépôts</Text>
                  <Pressable style={styles.secondaryBtnSmall} onPress={() => setActiveTab("ACCUEIL")}>
                    <Text style={styles.secondaryBtnText}>Retour</Text>
                  </Pressable>
                </View>

                <FlatList
                  data={deposits}
                  keyExtractor={(item) => item.id}
                  contentContainerStyle={{ paddingBottom: 24 }}
                  ListEmptyComponent={
                    <View style={styles.empty}>
                      <Text style={styles.emptyTitle}>Aucun dépôt</Text>
                      <Text style={styles.emptyText}>Crée ton premier dépôt depuis l’accueil.</Text>
                    </View>
                  }
                  renderItem={({ item }) => {
                    const ready = new Date(item.maturesAt).getTime() <= Date.now();
                    return (
                      <View style={styles.depositCard}>
                        <View style={{ flexDirection: "row", justifyContent: "space-between", gap: 12 }}>
                          <View style={{ flex: 1 }}>
                            <Text style={styles.depositTitle}>{item.composterName}</Text>
                            <Text style={styles.depositMeta}>
                              Déposé le {formatDateFR(item.createdAt)} • Prêt le {formatDateFR(item.maturesAt)}
                            </Text>
                          </View>
                          <View style={[styles.statusPill, ready ? styles.statusReady : styles.statusWait]}>
                            <Text style={styles.statusPillText}>{ready ? "✅ PRÊT" : "⏳ EN COURS"}</Text>
                          </View>
                        </View>

                        <View style={styles.depositRow}>
                          <View style={styles.kpiBox}>
                            <Text style={styles.kpiLabel}>Déchets</Text>
                            <Text style={styles.kpiValue}>{(item.wasteKg ?? 0).toFixed(2)} kg</Text>
                          </View>
                          <View style={styles.kpiBoxAccent}>
                            <Text style={styles.kpiLabel}>Compost (estim.)</Text>
                            <Text style={styles.kpiValue}>{(item.compostKg ?? 0).toFixed(2)} kg</Text>
                          </View>
                        </View>

                        <View style={{ marginTop: 12, flexDirection: "row", gap: 10 }}>
                          <Pressable style={styles.secondaryBtn} onPress={() => onDeleteDeposit(item.id)}>
                            <Text style={styles.secondaryBtnText}>🗑️ Supprimer</Text>
                          </Pressable>
                          <Pressable
                            style={styles.primaryBtn}
                            onPress={() => {
                              setSelectedComposterId(item.composterId);
                              setActiveTab("CARTE");
                            }}
                          >
                            <Text style={styles.primaryBtnText}>🗺️ Voir le point</Text>
                          </Pressable>
                        </View>
                      </View>
                    );
                  }}
                />
              </View>
            )}

            {activeTab === "CLASSEMENT" && (
              <View style={{ flex: 1 }}>
                <View style={styles.listTopBar}>
                  <Text style={styles.sectionTitle}>👑 Classement</Text>
                  <Pressable style={styles.secondaryBtnSmall} onPress={() => setActiveTab("ACCUEIL")}>
                    <Text style={styles.secondaryBtnText}>Retour</Text>
                  </Pressable>
                </View>

                <FlatList
                  data={leaderboard}
                  keyExtractor={(x) => x.id}
                  contentContainerStyle={{ paddingBottom: 24 }}
                  renderItem={({ item }) => {
                    const crown = item.rank === 1 ? "👑" : item.rank === 2 ? "🥈" : item.rank === 3 ? "🥉" : "";
                    const isMeRow = item.id === currentUserId;
                    return (
                      <View style={[styles.rankRow, isMeRow && styles.rankRowMe]}>
                        <Text style={styles.rankLeft}>
                          {item.rank}. {crown} {item.name}
                        </Text>
                        <Text style={styles.rankRight}>{item.points} pts</Text>
                      </View>
                    );
                  }}
                />
              </View>
            )}
          </>
        )}

        {/* ADMIN */}
        {isAdmin && (
          <ScrollView contentContainerStyle={{ paddingBottom: 24 }}>
            <View style={styles.card}>
              <View style={{ flexDirection: "row", justifyContent: "space-between", alignItems: "center" }}>
                <Text style={styles.cardTitle}>🛡️ Administration</Text>
                <Pressable style={styles.secondaryBtnSmall} onPress={signOutAdmin}>
                  <Text style={styles.secondaryBtnText}>Déconnexion</Text>
                </Pressable>
              </View>

              <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10, marginTop: 12 }}>
                <Pressable style={[styles.chip, adminTab === "DASH" && styles.chipActive]} onPress={() => setAdminTab("DASH")}>
                  <Text style={[styles.chipText, adminTab === "DASH" && styles.chipTextActive]}>➕ Ajout</Text>
                </Pressable>

                <Pressable style={[styles.chip, adminTab === "PILOTAGE" && styles.chipActive]} onPress={() => setAdminTab("PILOTAGE")}>
                  <Text style={[styles.chipText, adminTab === "PILOTAGE" && styles.chipTextActive]}>📡 Pilotage</Text>
                </Pressable>

                <Pressable style={[styles.chip, adminTab === "COMPOSTEURS" && styles.chipActive]} onPress={() => setAdminTab("COMPOSTEURS")}>
                  <Text style={[styles.chipText, adminTab === "COMPOSTEURS" && styles.chipTextActive]}>🗑️ Composteurs</Text>
                </Pressable>
              </ScrollView>
            </View>

            {adminTab === "DASH" && (
              <View style={styles.card}>
                <Text style={styles.cardTitle}>➕ Ajouter un composteur</Text>
                <Text style={styles.cardMuted}>Coordonnées en degrés décimaux.</Text>

                <Text style={styles.label}>Nom</Text>
                <TextInput value={aName} onChangeText={setAName} placeholder="Nom du point" placeholderTextColor={placeholderColor} style={styles.input} />

                <Text style={styles.label}>Adresse</Text>
                <TextInput value={aAddress} onChangeText={setAAddress} placeholder="Adresse / repère" placeholderTextColor={placeholderColor} style={styles.input} />

                <View style={{ flexDirection: "row", gap: 10 }}>
                  <View style={{ flex: 1 }}>
                    <Text style={styles.label}>Latitude</Text>
                    <TextInput value={aLat} onChangeText={setALat} placeholder="48.87" placeholderTextColor={placeholderColor} style={styles.input} keyboardType="numeric" />
                  </View>
                  <View style={{ flex: 1 }}>
                    <Text style={styles.label}>Longitude</Text>
                    <TextInput value={aLon} onChangeText={setALon} placeholder="2.22" placeholderTextColor={placeholderColor} style={styles.input} keyboardType="numeric" />
                  </View>
                </View>

                <View style={{ flexDirection: "row", gap: 10, marginTop: 12 }}>
                  <Pressable style={styles.primaryBtn} onPress={onAddComposter}>
                    <Text style={styles.primaryBtnText}>✅ Ajouter</Text>
                  </Pressable>
                </View>
              </View>
            )}

            {adminTab === "PILOTAGE" && (
              <View style={styles.card}>
                <Text style={styles.cardTitle}>📡 Pilotage à distance</Text>
                <Text style={styles.cardMuted}>
                  Modifie les paramètres (température, humidité, ventilateur, actions) puis envoie au composteur.
                </Text>

                <Text style={styles.label}>Composteur cible</Text>
                <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
                  {composters.map((c) => {
                    const active = c.id === selectedComposterId;
                    return (
                      <Pressable
                        key={c.id}
                        onPress={() => setSelectedComposterId(c.id)}
                        style={[styles.chip, active && styles.chipActive]}
                      >
                        <Text style={[styles.chipText, active && styles.chipTextActive]} numberOfLines={1}>
                          📍 {c.name}
                        </Text>
                      </Pressable>
                    );
                  })}
                </ScrollView>

                <View style={{ height: 10 }} />

                <Text style={styles.label}>Mode</Text>
                <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
                  {VAT_MODES.map((m) => {
                    const active = remoteModeKey === m.key;
                    return (
                      <Pressable
                        key={m.key}
                        onPress={() => setRemoteModeKey(m.key)}
                        style={[styles.chip, active && styles.chipActive]}
                      >
                        <Text style={[styles.chipText, active && styles.chipTextActive]}>
                          {m.icon} {m.label}
                        </Text>
                      </Pressable>
                    );
                  })}
                </ScrollView>

                <View style={{ height: 12 }} />

                <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
                  <Pressable style={[styles.chip, remoteTab === "TEMP" && styles.chipActive]} onPress={() => setRemoteTab("TEMP")}>
                    <Text style={[styles.chipText, remoteTab === "TEMP" && styles.chipTextActive]}>🌡️ Température</Text>
                  </Pressable>
                  <Pressable style={[styles.chip, remoteTab === "HUMID" && styles.chipActive]} onPress={() => setRemoteTab("HUMID")}>
                    <Text style={[styles.chipText, remoteTab === "HUMID" && styles.chipTextActive]}>💧 Humidité</Text>
                  </Pressable>
                  <Pressable style={[styles.chip, remoteTab === "FAN" && styles.chipActive]} onPress={() => setRemoteTab("FAN")}>
                    <Text style={[styles.chipText, remoteTab === "FAN" && styles.chipTextActive]}>🌀 Ventilateur</Text>
                  </Pressable>
                  <Pressable style={[styles.chip, remoteTab === "ACTION" && styles.chipActive]} onPress={() => setRemoteTab("ACTION")}>
                    <Text style={[styles.chipText, remoteTab === "ACTION" && styles.chipTextActive]}>🔁 Action</Text>
                  </Pressable>
                </ScrollView>

                <View style={{ height: 12 }} />

                {(() => {
                  const cfg = remoteConfig[remoteModeKey] || {};

                  if (remoteTab === "TEMP") {
                    return (
                      <View>
                        <Text style={styles.cardTitle}>🌡️ Température cible</Text>
                        <View style={{ flexDirection: "row", gap: 10 }}>
                          <View style={{ flex: 1 }}>
                            <Text style={styles.label}>Min (°C)</Text>
                            <TextInput
                              value={String(cfg.tempMin ?? "")}
                              onChangeText={(t) => patchRemote(remoteModeKey, { tempMin: Number(String(t).replace(",", ".")) || 0 })}
                              placeholder="Ex: 25"
                              placeholderTextColor={placeholderColor}
                              style={styles.input}
                              keyboardType="numeric"
                            />
                          </View>
                          <View style={{ flex: 1 }}>
                            <Text style={styles.label}>Max (°C)</Text>
                            <TextInput
                              value={String(cfg.tempMax ?? "")}
                              onChangeText={(t) => patchRemote(remoteModeKey, { tempMax: Number(String(t).replace(",", ".")) || 0 })}
                              placeholder="Ex: 40"
                              placeholderTextColor={placeholderColor}
                              style={styles.input}
                              keyboardType="numeric"
                            />
                          </View>
                        </View>
                      </View>
                    );
                  }

                  if (remoteTab === "HUMID") {
                    return (
                      <View>
                        <Text style={styles.cardTitle}>💧 Humidité cible</Text>
                        <Text style={styles.label}>Humidité (%)</Text>
                        <TextInput
                          value={String(cfg.humidity ?? "")}
                          onChangeText={(t) => patchRemote(remoteModeKey, { humidity: Number(String(t).replace(",", ".")) || 0 })}
                          placeholder="Ex: 70"
                          placeholderTextColor={placeholderColor}
                          style={styles.input}
                          keyboardType="numeric"
                        />
                      </View>
                    );
                  }

                  if (remoteTab === "FAN") {
                    return (
                      <View>
                        <Text style={styles.cardTitle}>🌀 Action ventilateur</Text>

                        <Text style={styles.label}>Mode ventilateur</Text>
                        <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={{ gap: 10 }}>
                          {FAN_MODES.map((m) => {
                            const active = cfg.fanMode === m.key;
                            return (
                              <Pressable
                                key={m.key}
                                onPress={() => patchRemote(remoteModeKey, { fanMode: m.key })}
                                style={[styles.chip, active && styles.chipActive]}
                              >
                                <Text style={[styles.chipText, active && styles.chipTextActive]}>{m.label}</Text>
                              </Pressable>
                            );
                          })}
                        </ScrollView>

                        <View style={{ height: 8 }} />

                        <View style={{ flexDirection: "row", gap: 10 }}>
                          <View style={{ flex: 1 }}>
                            <Text style={styles.label}>Seuil régulation (°C)</Text>
                            <TextInput
                              value={String(cfg.fanThresholdTemp ?? 65)}
                              onChangeText={(t) => patchRemote(remoteModeKey, { fanThresholdTemp: Number(String(t).replace(",", ".")) || 0 })}
                              placeholder="Ex: 65"
                              placeholderTextColor={placeholderColor}
                              style={styles.input}
                              keyboardType="numeric"
                            />
                          </View>
                          <View style={{ flex: 1 }}>
                            <Text style={styles.label}>Intervalle (heures)</Text>
                            <TextInput
                              value={String(cfg.fanIntervalHours ?? 2)}
                              onChangeText={(t) => patchRemote(remoteModeKey, { fanIntervalHours: Number(String(t).replace(",", ".")) || 0 })}
                              placeholder="Ex: 2"
                              placeholderTextColor={placeholderColor}
                              style={styles.input}
                              keyboardType="numeric"
                            />
                          </View>
                        </View>
                      </View>
                    );
                  }

                  return (
                    <View>
                      <Text style={styles.cardTitle}>🔁 Action physique</Text>
                      <Text style={styles.label}>Instruction</Text>
                      <TextInput
                        value={String(cfg.physicalAction ?? "")}
                        onChangeText={(t) => patchRemote(remoteModeKey, { physicalAction: t })}
                        placeholder="Ex: Mélange / Transfert / Récupération..."
                        placeholderTextColor={placeholderColor}
                        style={[styles.input, { height: 90 }]}
                        multiline
                      />
                    </View>
                  );
                })()}

                <View style={{ height: 12 }} />

                <View style={{ flexDirection: "row", gap: 10 }}>
                  <Pressable
                    style={styles.primaryBtn}
                    onPress={() => {
                      if (!selectedComposterId) return Alert.alert("Erreur", "Sélectionne un composteur.");
                      sendRemoteConfigToComposter(selectedComposterId);
                    }}
                  >
                    <Text style={styles.primaryBtnText}>📡 Envoyer au composteur</Text>
                  </Pressable>
                </View>

                {lastPush ? (
                  <Text style={[styles.cardMuted, { marginTop: 10 }]}>
                    Dernier envoi : {lastPush.composterName} • {formatDateFR(lastPush.sentAt)}
                  </Text>
                ) : (
                  <Text style={[styles.cardMuted, { marginTop: 10 }]}>Aucun envoi pour l’instant.</Text>
                )}
              </View>
            )}

            {adminTab === "COMPOSTEURS" && (
              <View style={styles.card}>
                <Text style={styles.cardTitle}>🗑️ Gestion des composteurs</Text>
                <Text style={styles.cardMuted}>Supprime un point si nécessaire.</Text>

                <View style={{ height: 10 }} />

                {composters.length === 0 ? (
                  <Text style={styles.cardMuted}>Aucun composteur.</Text>
                ) : (
                  composters.map((c) => (
                    <View key={c.id} style={styles.rowItem}>
                      <View style={{ flex: 1 }}>
                        <Text style={styles.rowTitle}>📍 {c.name}</Text>
                        <Text style={styles.rowSub}>
                          {c.address} • {c.latitude.toFixed(5)}, {c.longitude.toFixed(5)}
                        </Text>
                      </View>

                      <Pressable
                        style={styles.secondaryBtnSmall}
                        onPress={() => {
                          setSelectedComposterId(c.id);
                          setAdminTab("PILOTAGE");
                        }}
                      >
                        <Text style={styles.secondaryBtnText}>📡 Pilotage</Text>
                      </Pressable>

                      <View style={{ width: 8 }} />

                      <Pressable
                        style={[styles.secondaryBtnSmall, { borderColor: "rgba(255,42,42,0.35)" }]}
                        onPress={() => onDeleteComposter(c.id)}
                      >
                        <Text style={[styles.secondaryBtnText, { color: THEME.red }]}>🗑️</Text>
                      </Pressable>
                    </View>
                  ))
                )}
              </View>
            )}
          </ScrollView>
        )}
      </View>

      {/* USER TABS */}
      {!isAdmin ? (
        <View style={styles.tabs}>
          <Tab label="Accueil" active={activeTab === "ACCUEIL"} onPress={() => setActiveTab("ACCUEIL")} styles={styles} />
          <Tab label="Carte" active={activeTab === "CARTE"} onPress={() => setActiveTab("CARTE")} styles={styles} />
          <Tab label="Dépôts" active={activeTab === "DEPOTS"} onPress={() => setActiveTab("DEPOTS")} styles={styles} />
          <Tab label="👑" active={activeTab === "CLASSEMENT"} onPress={() => setActiveTab("CLASSEMENT")} styles={styles} />
        </View>
      ) : null}

      {/* DRAWER OVERLAY */}
      {drawerOpen ? (
        <Animated.View style={[styles.drawerOverlay, { opacity: overlayA }]}>
          <Pressable style={{ flex: 1 }} onPress={closeDrawer} />
        </Animated.View>
      ) : null}

      {/* DRAWER */}
      {drawerOpen ? (
        <Animated.View style={[styles.drawer, { transform: [{ translateX: drawerX }] }]}>
          <View style={styles.drawerHeader}>
            <Image source={LOGO} style={styles.drawerLogo} resizeMode="contain" />
            <Text style={styles.drawerTitle}>GreenLoop</Text>
            <Text style={styles.drawerSub}>
              {profile.firstName?.trim() ? profile.firstName.trim() : "Profil"} • {totalPoints} pts
            </Text>
          </View>

          {drawerPage === "HOME" && (
            <View style={{ padding: 14, gap: 10 }}>
              <Pressable
                style={styles.drawerItem}
                onPress={() => {
                  closeDrawer();
                  setScreen("IDENTITYQR");
                }}
              >
                <Text style={styles.drawerItemText}>🧾 S’identifier & QR code</Text>
              </Pressable>

              <Pressable style={styles.drawerItem} onPress={() => setDrawerPage("SETTINGS")}>
                <Text style={styles.drawerItemText}>⚙️ Paramètres</Text>
              </Pressable>

              {/* ✅ Nouvelle page utilisateur */}
              <Pressable
                style={styles.drawerItem}
                onPress={() => {
                  closeDrawer();
                  setScreen("LIVE");
                }}
              >
                <Text style={styles.drawerItemText}>📟 Suivi en temps réel</Text>
              </Pressable>

              <Pressable
                style={styles.drawerItem}
                onPress={() => {
                  closeDrawer();
                  setScreen("BOUTIQUE");
                }}
              >
                <Text style={styles.drawerItemText}>🎁 Boutique</Text>
              </Pressable>

              <Pressable
                style={styles.drawerItem}
                onPress={() => {
                  closeDrawer();
                  setScreen("IMPACT");
                }}
              >
                <Text style={styles.drawerItemText}>🌿 Impact écologique</Text>
              </Pressable>

              <Pressable style={styles.drawerItem} onPress={() => setDrawerPage("ADMIN")}>
                <Text style={styles.drawerItemText}>🛡️ Admin</Text>
              </Pressable>
            </View>
          )}

          

{drawerPage === "SETTINGS" && (
            <ScrollView contentContainerStyle={{ padding: 14, paddingBottom: 30 }}>
              <View style={styles.drawerBackRow}>
                <Pressable style={styles.secondaryBtnSmall} onPress={() => setDrawerPage("HOME")}>
                  <Text style={styles.secondaryBtnText}>Retour</Text>
                </Pressable>
              </View>

              <Text style={styles.drawerPageTitle}>⚙️ Paramètres</Text>
              <Text style={[styles.cardMuted, { marginTop: 6 }]}>Choisis le thème :</Text>

              <View style={{ flexDirection: "row", gap: 10, marginTop: 12 }}>
                <Pressable
                  style={[styles.modeBtn, themeMode === "dark" && styles.modeBtnActive]}
                  onPress={() => setThemeMode("dark")}
                >
                  <Text style={[styles.modeBtnText, themeMode === "dark" && styles.modeBtnTextActive]}>🌙 Sombre</Text>
                </Pressable>

                <Pressable
                  style={[styles.modeBtn, themeMode === "light" && styles.modeBtnActive]}
                  onPress={() => setThemeMode("light")}
                >
                  <Text style={[styles.modeBtnText, themeMode === "light" && styles.modeBtnTextActive]}>☀️ Clair</Text>
                </Pressable>
              </View>
            </ScrollView>
          )}

          {drawerPage === "ADMIN" && (
            <ScrollView contentContainerStyle={{ padding: 14, paddingBottom: 30 }}>
              <View style={styles.drawerBackRow}>
                <Pressable style={styles.secondaryBtnSmall} onPress={() => setDrawerPage("HOME")}>
                  <Text style={styles.secondaryBtnText}>Retour</Text>
                </Pressable>
              </View>

              <Text style={styles.drawerPageTitle}>🛡️ Admin</Text>

              {!isAdmin ? (
                <>
                  <Text style={[styles.cardMuted, { marginTop: 6 }]}>Entre le PIN pour accéder.</Text>

                  <Text style={styles.label}>PIN</Text>
                  <TextInput
                    value={pinInput}
                    onChangeText={setPinInput}
                    placeholder="Code admin"
                    placeholderTextColor={placeholderColor}
                    style={styles.input}
                    secureTextEntry
                  />

                  <Pressable style={[styles.primaryBtn, { marginTop: 12 }]} onPress={enterAdminFromDrawer}>
                    <Text style={styles.primaryBtnText}>➡️ Entrer</Text>
                  </Pressable>
                </>
              ) : (
                <>
                  <Text style={[styles.cardMuted, { marginTop: 6 }]}>✅ Déjà connecté en admin.</Text>
                  <Pressable style={[styles.secondaryBtn, { marginTop: 12 }]} onPress={signOutAdmin}>
                    <Text style={styles.secondaryBtnText}>Déconnexion</Text>
                  </Pressable>
                </>
              )}
            </ScrollView>
          )}
        </Animated.View>
      ) : null}
    </SafeAreaView>
  );
}

/* =========================
   SMALL COMPONENTS
========================= */
function Metric({ label, value, styles }) {
  return (
    <View style={styles.metric}>
      <Text style={styles.metricLabel}>{label}</Text>
      <Text style={styles.metricValue}>{value}</Text>
    </View>
  );
}

function Tab({ label, active, onPress, styles }) {
  return (
    <Pressable onPress={onPress} style={[styles.tab, active && styles.tabActive]}>
      <Text style={[styles.tabText, active && styles.tabTextActive]}>{label}</Text>
    </Pressable>
  );
}

/* =========================
   STYLES
========================= */
function makeStyles(THEME) {
  const borderSoft = THEME.mode === "dark" ? "rgba(255,255,255,0.18)" : "rgba(0,0,0,0.18)";
  const surface = THEME.mode === "dark" ? "#0F0F15" : "#F1F3F6";

  return {
    app: { flex: 1, backgroundColor: THEME.bg },

    boot: { flex: 1, alignItems: "center", justifyContent: "center" },
    bootLogo: { width: 140, height: 140 },
    bootTitle: { marginTop: 10, color: THEME.text, fontWeight: "900", fontSize: 22 },
    bootSub: { marginTop: 6, color: THEME.mutetext, fontWeight: "800" },
    bootBarOuter: {
      marginTop: 16,
      width: Math.min(260, SCREEN_W * 0.7),
      height: 10,
      borderRadius: 999,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.10)" : "rgba(0,0,0,0.08)",
      overflow: "hidden",
      borderWidth: 1,
      borderColor: THEME.line,
    },
    bootBarInner: { height: "100%", borderRadius: 999, backgroundColor: THEME.yellow },
    bootHint: { marginTop: 10, color: THEME.mutetext, fontWeight: "800", fontSize: 12 },

    header: {
      paddingHorizontal: 16,
      paddingTop: 10,
      paddingBottom: 10,
      borderBottomWidth: 1,
      borderBottomColor: THEME.line,
      flexDirection: "row",
      alignItems: "center",
      backgroundColor: THEME.bg,
    },
    burgerBtn: {
      width: 44,
      height: 44,
      borderRadius: 12,
      borderWidth: 1,
      borderColor: THEME.line,
      alignItems: "center",
      justifyContent: "center",
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.04)" : "rgba(0,0,0,0.03)",
    },
    bLine: { width: 18, height: 2, backgroundColor: THEME.text, borderRadius: 2, marginVertical: 2 },

    headerCenter: { flex: 1, alignItems: "center", justifyContent: "center", gap: 2 },
    headerLogo: { width: 36, height: 36 },
    headerTitleSmall: { color: THEME.text, fontSize: 14, fontWeight: "900" },
    headerSubtitleSmall: { color: THEME.mutetext, fontSize: 11, fontWeight: "800" },

    pointsPill: {
      flexDirection: "row",
      alignItems: "flex-end",
      gap: 6,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.14)" : "rgba(255,209,0,0.24)",
      borderWidth: 1,
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.28)" : "rgba(255,209,0,0.32)",
      borderRadius: 999,
      paddingHorizontal: 12,
      paddingVertical: 8,
    },
    pointsTxt: { color: THEME.text, fontWeight: "900", fontSize: 18 },
    pointsSmall: { color: THEME.mutetext, fontWeight: "900", fontSize: 11, paddingBottom: 2 },

    content: { flex: 1, paddingHorizontal: 16, paddingTop: 14 },

    hero: {
      backgroundColor: THEME.card,
      borderRadius: 20,
      padding: 14,
      borderWidth: 1,
      borderColor: THEME.line,
      shadowColor: THEME.shadow,
      shadowOpacity: 1,
      shadowRadius: 18,
      shadowOffset: { width: 0, height: 10 },
      elevation: 6,
    },
    heroTop: { flexDirection: "row", justifyContent: "space-between", alignItems: "center" },
    heroKicker: { color: THEME.text, fontWeight: "900", fontSize: 14 },
    heroBadge: {
      backgroundColor: THEME.mode === "dark" ? "rgba(255,42,42,0.12)" : "rgba(255,42,42,0.10)",
      borderColor: THEME.mode === "dark" ? "rgba(255,42,42,0.25)" : "rgba(255,42,42,0.20)",
      borderWidth: 1,
      paddingHorizontal: 10,
      paddingVertical: 6,
      borderRadius: 999,
    },
    heroBadgeText: { color: THEME.red, fontWeight: "900", fontSize: 12 },

    metricsRow: { flexDirection: "row", gap: 10, marginTop: 12 },
    metric: {
      flex: 1,
      backgroundColor: THEME.card2,
      borderRadius: 16,
      paddingVertical: 12,
      paddingHorizontal: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    metricLabel: { color: THEME.mutetext, fontSize: 11, fontWeight: "800" },
    metricValue: { color: THEME.text, fontSize: 18, fontWeight: "900", marginTop: 6 },

    card: {
      marginTop: 14,
      backgroundColor: THEME.card,
      borderRadius: 20,
      padding: 14,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    cardTitle: { color: THEME.text, fontWeight: "900", fontSize: 14, marginBottom: 8 },
    cardMain: { color: THEME.text, fontWeight: "900", fontSize: 16, marginTop: 2 },
    cardMuted: { color: THEME.mutetext, fontWeight: "700", fontSize: 12, marginTop: 2, lineHeight: 16 },

    label: { color: THEME.mutetext, fontWeight: "800", fontSize: 12, marginTop: 10, marginBottom: 6 },
    input: {
      backgroundColor: surface,
      borderWidth: 1,
      borderColor: THEME.line,
      borderRadius: 14,
      paddingHorizontal: 12,
      paddingVertical: 12,
      color: THEME.text,
      fontWeight: "800",
    },

    chip: {
      maxWidth: SCREEN_W * 0.7,
      backgroundColor: surface,
      borderWidth: 1,
      borderColor: THEME.line,
      paddingHorizontal: 12,
      paddingVertical: 10,
      borderRadius: 999,
    },
    chipActive: {
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.10)" : "rgba(255,209,0,0.16)",
      borderColor: "rgba(255,209,0,0.35)",
    },
    chipText: { color: THEME.text, fontWeight: "800", fontSize: 12 },
    chipTextActive: { color: THEME.yellow, fontWeight: "900" },

    primaryBtn: {
      flex: 1,
      backgroundColor: THEME.yellow,
      borderRadius: 14,
      paddingVertical: 12,
      alignItems: "center",
      justifyContent: "center",
    },
    primaryBtnText: { color: "#111114", fontWeight: "900" },

    secondaryBtn: {
      flex: 1,
      backgroundColor: "transparent",
      borderRadius: 14,
      paddingVertical: 12,
      alignItems: "center",
      justifyContent: "center",
      borderWidth: 1,
      borderColor: borderSoft,
    },
    secondaryBtnSmall: {
      backgroundColor: "transparent",
      borderRadius: 12,
      paddingVertical: 10,
      paddingHorizontal: 12,
      alignItems: "center",
      justifyContent: "center",
      borderWidth: 1,
      borderColor: borderSoft,
    },
    secondaryBtnText: { color: THEME.text, fontWeight: "900" },

    modeBtn: {
      flex: 1,
      borderRadius: 14,
      paddingVertical: 12,
      alignItems: "center",
      justifyContent: "center",
      borderWidth: 1,
      borderColor: borderSoft,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.04)" : "rgba(0,0,0,0.03)",
    },
    modeBtnActive: {
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.35)" : "rgba(255,209,0,0.35)",
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.10)" : "rgba(255,209,0,0.14)",
    },
    modeBtnText: { color: THEME.text, fontWeight: "900" },
    modeBtnTextActive: { color: THEME.yellow },


    infoStrip: {
      marginTop: 12,
      backgroundColor: surface,
      borderWidth: 1,
      borderColor: THEME.line,
      borderRadius: 14,
      paddingHorizontal: 12,
      paddingVertical: 10,
    },
    infoStripText: { color: THEME.mutetext, fontWeight: "800", fontSize: 12, lineHeight: 16 },

    map: { flex: 1, borderRadius: 18, overflow: "hidden" },
    mapTopBar: { flexDirection: "row", gap: 10, marginBottom: 10 },
    mapBottomSheet: {
      position: "absolute",
      left: 16,
      right: 16,
      bottom: 16,
      backgroundColor: THEME.card,
      borderRadius: 20,
      padding: 14,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    sheetTitle: { color: THEME.mutetext, fontWeight: "900", fontSize: 12 },
    sheetMain: { color: THEME.text, fontWeight: "900", fontSize: 16, marginTop: 6 },
    sheetMuted: { color: THEME.mutetext, fontWeight: "700", fontSize: 12, marginTop: 2 },

    pin: {
      width: 26,
      height: 26,
      borderRadius: 13,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.10)" : "rgba(0,0,0,0.08)",
      borderWidth: 1,
      borderColor: THEME.mode === "dark" ? "rgba(255,255,255,0.20)" : "rgba(0,0,0,0.20)",
      alignItems: "center",
      justifyContent: "center",
    },
    pinActive: {
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.18)" : "rgba(255,209,0,0.22)",
      borderColor: "rgba(255,209,0,0.35)",
    },
    pinDot: { width: 10, height: 10, borderRadius: 5, backgroundColor: THEME.yellow },

    listTopBar: { flexDirection: "row", justifyContent: "space-between", alignItems: "center", marginBottom: 10 },
    sectionTitle: { color: THEME.text, fontWeight: "900", fontSize: 16 },

    depositCard: {
      marginBottom: 12,
      backgroundColor: THEME.card,
      borderRadius: 20,
      padding: 14,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    depositTitle: { color: THEME.text, fontWeight: "900", fontSize: 14 },
    depositMeta: { color: THEME.mutetext, fontWeight: "700", fontSize: 12, marginTop: 4 },
    depositRow: { flexDirection: "row", gap: 10, marginTop: 12 },
    kpiBox: {
      flex: 1,
      backgroundColor: surface,
      borderRadius: 16,
      padding: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    kpiBoxAccent: {
      flex: 1,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.08)" : "rgba(255,209,0,0.14)",
      borderRadius: 16,
      padding: 12,
      borderWidth: 1,
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.22)" : "rgba(255,209,0,0.28)",
    },
    kpiLabel: { color: THEME.mutetext, fontSize: 11, fontWeight: "800" },
    kpiValue: { color: THEME.text, fontSize: 14, fontWeight: "900", marginTop: 6 },

    statusPill: { borderRadius: 999, paddingHorizontal: 10, paddingVertical: 6, borderWidth: 1, alignSelf: "flex-start" },
    statusReady: { backgroundColor: "rgba(55,214,122,0.12)", borderColor: "rgba(55,214,122,0.25)" },
    statusWait: {
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.06)" : "rgba(0,0,0,0.06)",
      borderColor: THEME.mode === "dark" ? "rgba(255,255,255,0.14)" : "rgba(0,0,0,0.14)",
    },
    statusPillText: { color: THEME.text, fontWeight: "900", fontSize: 11 },

    empty: {
      marginTop: 40,
      padding: 18,
      backgroundColor: THEME.card,
      borderRadius: 20,
      borderWidth: 1,
      borderColor: THEME.line,
      alignItems: "center",
    },
    emptyTitle: { color: THEME.text, fontWeight: "900", fontSize: 16 },
    emptyText: { color: THEME.mutetext, fontWeight: "700", fontSize: 12, marginTop: 8, textAlign: "center" },

    rankRow: {
      marginHorizontal: 16,
      marginBottom: 10,
      padding: 14,
      backgroundColor: THEME.card,
      borderRadius: 18,
      borderWidth: 1,
      borderColor: THEME.line,
      flexDirection: "row",
      justifyContent: "space-between",
      alignItems: "center",
    },
    rankRowMe: {
      borderColor: "rgba(255,209,0,0.35)",
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.10)" : "rgba(255,209,0,0.14)",
    },
    rankLeft: { color: THEME.text, fontWeight: "900" },
    rankRight: { color: THEME.yellow, fontWeight: "900" },

    tabs: {
      flexDirection: "row",
      borderTopWidth: 1,
      borderTopColor: THEME.line,
      backgroundColor: THEME.bg,
      paddingHorizontal: 10,
      paddingVertical: 10,
      gap: 10,
    },
    tab: {
      flex: 1,
      backgroundColor: "transparent",
      borderRadius: 14,
      borderWidth: 1,
      borderColor: THEME.mode === "dark" ? "rgba(255,255,255,0.12)" : "rgba(0,0,0,0.12)",
      paddingVertical: 10,
      alignItems: "center",
      justifyContent: "center",
    },
    tabActive: {
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.12)" : "rgba(255,209,0,0.18)",
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.25)" : "rgba(255,209,0,0.30)",
    },
    tabText: { color: THEME.mutetext, fontWeight: "900", fontSize: 12 },
    tabTextActive: { color: THEME.yellow },

    drawerOverlay: { position: "absolute", left: 0, top: 0, right: 0, bottom: 0, backgroundColor: THEME.overlay },
    drawer: {
      position: "absolute",
      top: 0,
      bottom: 0,
      left: 0,
      width: Math.min(340, SCREEN_W * 0.86),
      backgroundColor: THEME.card,
      borderRightWidth: 1,
      borderRightColor: THEME.line,
      paddingTop: 12,
      elevation: 12,
      shadowColor: "#000",
      shadowOpacity: 0.25,
      shadowRadius: 20,
      shadowOffset: { width: 6, height: 10 },
    },
    drawerHeader: { paddingHorizontal: 14, paddingBottom: 12, borderBottomWidth: 1, borderBottomColor: THEME.line, gap: 6 },
    drawerLogo: { width: 54, height: 54 },
    drawerTitle: { color: THEME.text, fontWeight: "900", fontSize: 18 },
    drawerSub: { color: THEME.mutetext, fontWeight: "800", fontSize: 12 },

    drawerItem: {
      paddingVertical: 14,
      paddingHorizontal: 12,
      borderRadius: 14,
      borderWidth: 1,
      borderColor: THEME.line,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.04)" : "rgba(0,0,0,0.03)",
    },
    drawerItemText: { color: THEME.text, fontWeight: "900" },
    drawerBackRow: { marginBottom: 10 },
    drawerPageTitle: { color: THEME.text, fontWeight: "900", fontSize: 18, marginBottom: 6 },

    rowItem: { flexDirection: "row", alignItems: "center", gap: 10, paddingVertical: 10, borderTopWidth: 1, borderTopColor: THEME.line },
    rowTitle: { color: THEME.text, fontWeight: "900", fontSize: 13 },
    rowSub: { color: THEME.mutetext, fontWeight: "700", fontSize: 12, marginTop: 2 },

    // Impact header styles reused for LIVE back button
    impactHeader: {
      paddingHorizontal: 16,
      paddingTop: 12,
      paddingBottom: 12,
      borderBottomWidth: 1,
      borderBottomColor: THEME.line,
      flexDirection: "row",
      alignItems: "center",
      gap: 12,
    },
    impactBackBtn: {
      paddingVertical: 10,
      paddingHorizontal: 12,
      borderRadius: 12,
      borderWidth: 1,
      borderColor: borderSoft,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.05)" : "rgba(0,0,0,0.03)",
    },
    impactBackTxt: { color: THEME.text, fontWeight: "900" },
    impactTitle: { color: THEME.text, fontWeight: "900", fontSize: 18 },
    impactSub: { color: THEME.mutetext, fontWeight: "800", fontSize: 12, marginTop: 2 },

    impactBigCard: {
      marginTop: 14,
      marginHorizontal: 16,
      padding: 16,
      borderRadius: 20,
      borderWidth: 1,
      borderColor: THEME.line,
      backgroundColor: THEME.mode === "dark" ? "#0F1A14" : "#FFFFFF",
    },
    impactCardTitle: { color: THEME.text, fontWeight: "900", fontSize: 14 },
    impactBigNumber: { color: THEME.green, fontWeight: "900", fontSize: 36, marginTop: 10 },
    impactCardDesc: { color: THEME.mutetext, fontWeight: "800", fontSize: 12, marginTop: 8, lineHeight: 16 },

    impactRow: { marginTop: 12, marginHorizontal: 16, flexDirection: "row", gap: 12 },
    impactMiniCard: {
      flex: 1,
      padding: 14,
      borderRadius: 18,
      borderWidth: 1,
      borderColor: THEME.line,
      backgroundColor: THEME.card,
    },
    impactMiniTitle: { color: THEME.mutetext, fontWeight: "900", fontSize: 12 },
    impactMiniValue: { color: THEME.text, fontWeight: "900", fontSize: 20, marginTop: 8 },

    // Live
    liveHeader: {
      paddingHorizontal: 16,
      paddingTop: 12,
      paddingBottom: 12,
      borderBottomWidth: 1,
      borderBottomColor: THEME.line,
      flexDirection: "row",
      alignItems: "center",
      gap: 12,
    },
    liveTitle: { color: THEME.text, fontWeight: "900", fontSize: 18 },
    liveSub: { color: THEME.mutetext, fontWeight: "800", fontSize: 12, marginTop: 2 },

    liveVatCard: {
      marginTop: 12,
      padding: 14,
      borderRadius: 18,
      borderWidth: 1,
      borderColor: THEME.line,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.03)" : "rgba(0,0,0,0.03)",
    },
    liveVatTitle: { color: THEME.text, fontWeight: "900", fontSize: 14 },
    liveModePill: {
      borderRadius: 999,
      paddingHorizontal: 10,
      paddingVertical: 6,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.10)" : "rgba(255,209,0,0.16)",
      borderWidth: 1,
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.25)" : "rgba(255,209,0,0.28)",
    },
    liveModePillText: { color: THEME.yellow, fontWeight: "900", fontSize: 12 },

    liveRow: { flexDirection: "row", gap: 10, marginTop: 12 },
    liveKpi: {
      flex: 1,
      backgroundColor: THEME.card,
      borderRadius: 16,
      padding: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    liveKpiWide: {
      flex: 1,
      backgroundColor: THEME.card,
      borderRadius: 16,
      padding: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    liveKpiLabel: { color: THEME.mutetext, fontWeight: "800", fontSize: 11 },
    liveKpiValue: { color: THEME.text, fontWeight: "900", fontSize: 18, marginTop: 6 },

    liveActionBox: {
      marginTop: 12,
      backgroundColor: THEME.card,
      borderRadius: 16,
      padding: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    liveActionTitle: { color: THEME.mutetext, fontWeight: "900", fontSize: 12 },
    liveActionText: { color: THEME.text, fontWeight: "800", fontSize: 12, marginTop: 8, lineHeight: 16 },


    liveTabsRow: { flexDirection: "row", gap: 10, marginTop: 12 },
    liveTabBtn: {
      flex: 1,
      borderRadius: 14,
      paddingVertical: 10,
      alignItems: "center",
      justifyContent: "center",
      borderWidth: 1,
      borderColor: borderSoft,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.03)" : "rgba(0,0,0,0.03)",
    },
    liveTabBtnActive: {
      borderColor: THEME.mode === "dark" ? "rgba(255,209,0,0.32)" : "rgba(255,209,0,0.32)",
      backgroundColor: THEME.mode === "dark" ? "rgba(255,209,0,0.10)" : "rgba(255,209,0,0.14)",
    },
    liveTabBtnText: { color: THEME.mutetext, fontWeight: "900", fontSize: 12 },
    liveTabBtnTextActive: { color: THEME.yellow },

    cuve4Big: { marginTop: 14, alignItems: "center" },
    cuve4BigValue: { color: THEME.yellow, fontWeight: "900", fontSize: 44 },
    cuve4BigLabel: { color: THEME.mutetext, fontWeight: "900", marginTop: 4 },
    cuve4BarOuter: {
      marginTop: 14,
      height: 12,
      borderRadius: 999,
      backgroundColor: THEME.mode === "dark" ? "rgba(255,255,255,0.10)" : "rgba(0,0,0,0.08)",
      overflow: "hidden",
      borderWidth: 1,
      borderColor: THEME.line,
    },
    cuve4BarInner: { height: "100%", borderRadius: 999, backgroundColor: THEME.yellow },

    // Boutique
    shopHeader: { backgroundColor: "#2BBF6A", paddingTop: 14, paddingBottom: 12, paddingHorizontal: 16 },
    shopHeaderTop: { flexDirection: "row", alignItems: "center", justifyContent: "space-between" },
    shopTitle: { color: "#fff", fontWeight: "900", fontSize: 28 },
    shopPointsPill: {
      backgroundColor: "rgba(255,255,255,0.95)",
      paddingHorizontal: 14,
      paddingVertical: 10,
      borderRadius: 999,
    },
    shopPointsText: { color: "#2BBF6A", fontWeight: "900" },

    shopTabsRow: { flexDirection: "row", gap: 16, marginTop: 14 },
    shopTab: { paddingVertical: 8 },
    shopTabActive: { borderBottomWidth: 3, borderBottomColor: "#fff" },
    shopTabText: { color: "rgba(255,255,255,0.75)", fontWeight: "900" },
    shopTabTextActive: { color: "#fff" },

    shopTopInfo: {
      backgroundColor: THEME.card,
      borderRadius: 18,
      borderWidth: 1,
      borderColor: THEME.line,
      padding: 14,
      marginBottom: 14,
    },
    shopBackBtn: {
      alignSelf: "flex-start",
      paddingVertical: 10,
      paddingHorizontal: 12,
      borderRadius: 12,
      borderWidth: 1,
      borderColor: THEME.line,
    },
    shopBackTxt: { color: THEME.text, fontWeight: "900" },
    shopInfoText: { color: THEME.mutetext, fontWeight: "800", lineHeight: 18 },
    shopInfoStrong: { color: THEME.text, fontWeight: "900" },

    shopCard: {
      flex: 1,
      backgroundColor: THEME.card,
      borderRadius: 18,
      borderWidth: 1,
      borderColor: THEME.line,
      padding: 12,
      marginBottom: 12,
    },
    shopImageWrap: {
      height: 130,
      borderRadius: 14,
      backgroundColor: THEME.mode === "dark" ? "rgba(43,191,106,0.18)" : "rgba(43,191,106,0.14)",
      alignItems: "center",
      justifyContent: "center",
      borderWidth: 1,
      borderColor: THEME.line,
    },
    shopImage: { width: "75%", height: "75%" },

    shopCardTitle: { marginTop: 10, color: THEME.text, fontWeight: "900", fontSize: 14 },

    shopCostRow: { flexDirection: "row", alignItems: "center", gap: 8, marginTop: 8 },
    shopCoin: { width: 18, height: 18, borderRadius: 9, backgroundColor: "#2BBF6A" },
    shopCost: { color: "#2BBF6A", fontWeight: "900", fontSize: 18 },

    shopHint: { marginTop: 6, color: THEME.mutetext, fontWeight: "800", fontSize: 12 },
  };
}
