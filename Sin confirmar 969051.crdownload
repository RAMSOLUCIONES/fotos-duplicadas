import React, { useState } from 'react';
import {
  View, Text, TouchableOpacity, FlatList, Image, StyleSheet, StatusBar,
  Modal, TextInput, Alert, ActivityIndicator, Dimensions, ScrollView,
} from 'react-native';
import * as MediaLibrary from 'expo-media-library';
import * as FileSystem from 'expo-file-system';

const C = {
  bg: '#eef1f4', card: '#ffffff', ink: '#16202a', muted: '#5d6b79', line: '#d9e0e7',
  accent: '#2c4fd8', keep: '#2b8a68', del: '#cf3f3f', move: '#c77a12',
};
const LABEL = { keep: 'Conservar', move: 'Mover', delete: 'Borrar' };
const W = Dimensions.get('window').width;
const TILE = Math.floor((W - 16 * 2 - 12 * 2 - 10) / 2);

const fmtSize = (b) => (b >= 1048576 ? (b / 1048576).toFixed(1) + ' MB' : Math.max(1, Math.round((b || 0) / 1024)) + ' KB');
const fmtDur = (s) => Math.floor(s / 60) + ':' + String(Math.round(s % 60)).padStart(2, '0');
const pause = () => new Promise((r) => setTimeout(r, 0));

// Lee tamaño (y opcionalmente huella MD5) de un archivo de la galería
async function readFile(asset, md5) {
  const opts = md5 ? { md5: true, size: true } : { size: true };
  try {
    const i = await FileSystem.getInfoAsync(asset.uri, opts);
    if (i.exists) return i;
  } catch (e) {}
  try {
    const info = await MediaLibrary.getAssetInfoAsync(asset);
    if (info.localUri) return await FileSystem.getInfoAsync(info.localUri, opts);
  } catch (e) {}
  return null;
}

// Busca archivos idénticos: 1) mismo tipo/medidas/duración, 2) mismo tamaño, 3) mismo contenido (MD5)
async function findDuplicates(assets, onProgress) {
  const buckets = {};
  assets.forEach((a) => {
    const k = [a.mediaType, a.width, a.height, Math.round(a.duration || 0)].join('|');
    (buckets[k] = buckets[k] || []).push(a);
  });
  const cands = [].concat(...Object.values(buckets).filter((b) => b.length > 1));

  const bySize = {};
  for (let i = 0; i < cands.length; i++) {
    const a = cands[i];
    const f = await readFile(a, false);
    if (f && f.size != null) {
      a.size = f.size;
      const k = a.mediaType + '|' + f.size;
      (bySize[k] = bySize[k] || []).push(a);
    }
    onProgress('Comparando tamaños', i + 1, cands.length);
    if (i % 10 === 9) await pause();
  }

  const second = Object.values(bySize).filter((b) => b.length > 1);
  const total = second.reduce((s, b) => s + b.length, 0);
  let done = 0;
  const byHash = {};
  for (const b of second) {
    for (const a of b) {
      const f = await readFile(a, true);
      if (f && f.md5) {
        const k = f.md5 + '|' + a.size;
        (byHash[k] = byHash[k] || []).push(a);
      }
      done++;
      onProgress('Comparando contenido', done, total);
      await pause();
    }
  }

  return Object.values(byHash)
    .filter((b) => b.length > 1)
    .map((items) => {
      items.sort((x, y) => (x.creationTime || 0) - (y.creationTime || 0));
      return { key: String(items[0].id), items };
    })
    .sort((a, b) => b.items.length - a.items.length);
}

function Btn({ title, onPress, kind, disabled }) {
  return (
    <TouchableOpacity
      onPress={onPress}
      disabled={disabled}
      style={[s.btn, kind === 'primary' && s.btnPrimary, disabled && { opacity: 0.4 }]}
    >
      <Text style={[s.btnTxt, kind === 'primary' && { color: '#fff' }]}>{title}</Text>
    </TouchableOpacity>
  );
}

function Seg({ value, onChange, big }) {
  return (
    <View style={s.seg}>
      {['keep', 'move', 'delete'].map((k) => {
        const on = value === k;
        return (
          <TouchableOpacity
            key={k}
            onPress={() => onChange(k)}
            style={[s.segBtn, big && { height: 48 }, on && { backgroundColor: C[k === 'delete' ? 'del' : k], borderColor: C[k === 'delete' ? 'del' : k] }]}
          >
            <Text style={[s.segTxt, on && { color: '#fff' }]}>{LABEL[k]}</Text>
          </TouchableOpacity>
        );
      })}
    </View>
  );
}

export default function App() {
  const [screen, setScreen] = useState('home'); // home | folders | scan | results
  const [busy, setBusy] = useState(false);
  const [folders, setFolders] = useState([]);
  const [folder, setFolder] = useState(null);
  const [scanned, setScanned] = useState(0);
  const [progress, setProgress] = useState({ label: '', i: 0, n: 0 });
  const [groups, setGroups] = useState([]);
  const [marks, setMarks] = useState({});
  const [viewer, setViewer] = useState(null); // {gi, k}
  const [review, setReview] = useState(false);
  const [dest, setDest] = useState('Duplicados');

  const st = (id) => marks[id] || 'keep';
  const setMark = (id, v) => setMarks((m) => ({ ...m, [id]: v }));

  async function openFolders() {
    const p = await MediaLibrary.requestPermissionsAsync(false, ['photo', 'video']);
    if (!p.granted) {
      Alert.alert('Falta el permiso', 'Permite el acceso a fotos y videos en Ajustes > Apps > Fotos Duplicadas > Permisos.');
      return;
    }
    setBusy(true);
    try {
      const list = await MediaLibrary.getAlbumsAsync();
      list.sort((a, b) => b.assetCount - a.assetCount);
      setFolders(list);
      setScreen('folders');
    } catch (e) {
      Alert.alert('Error', String((e && e.message) || e));
    }
    setBusy(false);
  }

  async function scanFolder(album) {
    setFolder(album);
    setScreen('scan');
    setProgress({ label: 'Leyendo carpeta', i: 0, n: album ? album.assetCount : 0 });
    try {
      let all = [];
      let after;
      for (;;) {
        const page = await MediaLibrary.getAssetsAsync({
          album: album ? album.id : undefined,
          first: 500,
          after,
          mediaType: ['photo', 'video'],
        });
        all = all.concat(page.assets);
        setProgress({ label: 'Leyendo carpeta', i: all.length, n: album ? album.assetCount : 0 });
        if (!page.hasNextPage) break;
        after = page.endCursor;
      }
      setScanned(all.length);
      const g = await findDuplicates(all, (label, i, n) => setProgress({ label, i, n }));
      setGroups(g);
      setMarks({});
      setScreen('results');
    } catch (e) {
      Alert.alert('Error', String((e && e.message) || e));
      setScreen('folders');
    }
  }

  function autoMark() {
    const m = {};
    groups.forEach((g) => g.items.forEach((a, k) => { if (k > 0) m[a.id] = 'delete'; }));
    setMarks(m);
  }

  const all = [].concat(...groups.map((g) => g.items));
  const toDelete = all.filter((a) => st(a.id) === 'delete');
  const toMove = all.filter((a) => st(a.id) === 'move');
  const freed = toDelete.reduce((t, a) => t + (a.size || 0), 0);
  const extra = groups.reduce((t, g) => t + g.items.length - 1, 0);
  const recoverable = groups.reduce((t, g) => t + g.items.slice(1).reduce((x, a) => x + (a.size || 0), 0), 0);

  async function apply() {
    const removed = new Set();
    try {
      if (toDelete.length) {
        const ok = await MediaLibrary.deleteAssetsAsync(toDelete);
        if (ok) toDelete.forEach((a) => removed.add(a.id));
        else Alert.alert('Borrado cancelado', 'No se borró ninguna foto ni video.');
      }
    } catch (e) {
      Alert.alert('No se pudo borrar', String((e && e.message) || e));
    }
    try {
      if (toMove.length) {
        const name = dest.trim() || 'Duplicados';
        let album = await MediaLibrary.getAlbumAsync(name);
        if (!album) {
          album = await MediaLibrary.createAlbumAsync(name, toMove[0], false);
          if (toMove.length > 1) await MediaLibrary.addAssetsToAlbumAsync(toMove.slice(1), album, false);
        } else {
          await MediaLibrary.addAssetsToAlbumAsync(toMove, album, false);
        }
        toMove.forEach((a) => removed.add(a.id));
      }
    } catch (e) {
      Alert.alert('No se pudo mover', String((e && e.message) || e));
    }
    setGroups((gs) =>
      gs.map((g) => ({ ...g, items: g.items.filter((a) => !removed.has(a.id)) })).filter((g) => g.items.length > 1)
    );
    setMarks({});
    setReview(false);
    setViewer(null);
    if (removed.size) Alert.alert('Listo', removed.size + ' archivo(s) procesado(s).');
  }

  /* ---------- Pantallas ---------- */
  let body;
  if (screen === 'home') {
    body = (
      <View style={s.pad}>
        <Text style={s.h1}>Fotos duplicadas</Text>
        <Text style={s.lead}>
          Elige una carpeta de tu teléfono. Busco las fotos y videos repetidos y tú decides cuáles conservar, mover o borrar.
        </Text>
        <Btn title="Elegir carpeta" kind="primary" onPress={openFolders} disabled={busy} />
        {busy && <ActivityIndicator style={{ marginTop: 16 }} color={C.accent} />}
        <Text style={s.small}>Todo se analiza en tu teléfono. No se sube nada a internet.</Text>
      </View>
    );
  } else if (screen === 'folders') {
    body = (
      <FlatList
        data={folders}
        keyExtractor={(f) => String(f.id)}
        contentContainerStyle={s.pad}
        ListHeaderComponent={
          <View>
            <Text style={s.h1}>Elige una carpeta</Text>
            <TouchableOpacity style={[s.row, { borderColor: C.accent }]} onPress={() => scanFolder(null)}>
              <Text style={[s.rowTitle, { color: C.accent }]}>Toda la galería</Text>
              <Text style={s.rowSub}>Fotos y videos de todas las carpetas</Text>
            </TouchableOpacity>
          </View>
        }
        renderItem={({ item }) => (
          <TouchableOpacity style={s.row} onPress={() => scanFolder(item)}>
            <Text style={s.rowTitle}>{item.title}</Text>
            <Text style={s.rowSub}>{item.assetCount} archivos</Text>
          </TouchableOpacity>
        )}
        ListFooterComponent={<Btn title="Volver" onPress={() => setScreen('home')} />}
      />
    );
  } else if (screen === 'scan') {
    const pct = progress.n ? Math.min(100, Math.round((progress.i / progress.n) * 100)) : 0;
    body = (
      <View style={s.pad}>
        <Text style={s.h1}>Analizando…</Text>
        <Text style={s.lead}>{folder ? folder.title : 'Toda la galería'}</Text>
        <View style={s.bar}><View style={[s.barFill, { width: pct + '%' }]} /></View>
        <Text style={s.lead}>{progress.label}: {progress.i}{progress.n ? ' de ' + progress.n : ''}</Text>
      </View>
    );
  } else {
    body = (
      <FlatList
        data={groups}
        keyExtractor={(g) => g.key}
        contentContainerStyle={[s.pad, { paddingBottom: 120 }]}
        ListHeaderComponent={
          <View>
            <Text style={s.h1}>Resultados</Text>
            <Text style={s.lead}>{folder ? folder.title : 'Toda la galería'}</Text>
            <View style={s.stats}>
              <View style={s.stat}><Text style={s.statN}>{scanned}</Text><Text style={s.statL}>revisados</Text></View>
              <View style={s.stat}><Text style={s.statN}>{extra}</Text><Text style={s.statL}>repetidos</Text></View>
              <View style={s.stat}><Text style={s.statN}>{fmtSize(recoverable)}</Text><Text style={s.statL}>recuperables</Text></View>
            </View>
            <Btn title="Marcar duplicados (dejar el más antiguo)" onPress={autoMark} disabled={!groups.length} />
            <View style={{ height: 8 }} />
            <Btn title="Elegir otra carpeta" onPress={() => setScreen('folders')} />
            {!groups.length && (
              <Text style={[s.lead, { marginTop: 24, textAlign: 'center' }]}>No encontré duplicados en esta carpeta.</Text>
            )}
          </View>
        }
        renderItem={({ item: g, index: gi }) => (
          <View style={s.group}>
            <Text style={s.groupTitle}>{g.items.length} copias idénticas</Text>
            <View style={s.tiles}>
              {g.items.map((a, k) => {
                const v = st(a.id);
                return (
                  <View key={a.id} style={{ width: TILE }}>
                    <TouchableOpacity onPress={() => setViewer({ gi, k })}>
                      <Image source={{ uri: a.uri }} style={[s.thumb, v === 'delete' && { opacity: 0.45 }]} />
                      {a.mediaType === 'video' && (
                        <View style={s.vid}><Text style={s.vidTxt}>▶ {fmtDur(a.duration || 0)}</Text></View>
                      )}
                      <View style={[s.badge, { backgroundColor: C[v === 'delete' ? 'del' : v] }]}>
                        <Text style={s.badgeTxt}>{k === 0 && v === 'keep' ? 'Original' : LABEL[v]}</Text>
                      </View>
                    </TouchableOpacity>
                    <Text numberOfLines={1} style={s.name}>{a.filename}</Text>
                    <Text style={s.meta}>{a.width}×{a.height} · {fmtSize(a.size)}</Text>
                    <Seg value={v} onChange={(x) => setMark(a.id, x)} />
                  </View>
                );
              })}
            </View>
          </View>
        )}
      />
    );
  }

  const vItem = viewer && groups[viewer.gi] ? groups[viewer.gi].items[viewer.k] : null;

  return (
    <View style={s.root}>
      <StatusBar barStyle="dark-content" backgroundColor={C.bg} />
      {body}

      {screen === 'results' && (toDelete.length + toMove.length) > 0 && (
        <View style={s.dock}>
          <Text style={s.dockTxt}>
            {toDelete.length} para borrar · {toMove.length} para mover{toDelete.length ? '\nLiberarías ' + fmtSize(freed) : ''}
          </Text>
          <Btn title="Revisar" kind="primary" onPress={() => setReview(true)} />
        </View>
      )}

      {/* Visor */}
      <Modal visible={!!vItem} animationType="fade" onRequestClose={() => setViewer(null)}>
        {vItem && (
          <View style={s.viewer}>
            <View style={s.vTop}>
              <Text style={{ color: '#fff' }}>{viewer.k + 1} de {groups[viewer.gi].items.length}</Text>
              <TouchableOpacity onPress={() => setViewer(null)}><Text style={s.close}>Cerrar</Text></TouchableOpacity>
            </View>
            <Image source={{ uri: vItem.uri }} style={s.vImg} resizeMode="contain" />
            <Text style={s.vInfo}>
              {vItem.filename}{'\n'}{vItem.width}×{vItem.height} · {fmtSize(vItem.size)}
              {vItem.mediaType === 'video' ? ' · video ' + fmtDur(vItem.duration || 0) + '\n(para reproducirlo, ábrelo en tu galería)' : ''}
            </Text>
            <View style={s.vBar}>
              <TouchableOpacity
                style={[s.nav, viewer.k === 0 && { opacity: 0.3 }]}
                disabled={viewer.k === 0}
                onPress={() => setViewer({ gi: viewer.gi, k: viewer.k - 1 })}
              ><Text style={s.navTxt}>‹</Text></TouchableOpacity>
              <View style={{ flex: 1 }}><Seg big value={st(vItem.id)} onChange={(x) => setMark(vItem.id, x)} /></View>
              <TouchableOpacity
                style={[s.nav, viewer.k === groups[viewer.gi].items.length - 1 && { opacity: 0.3 }]}
                disabled={viewer.k === groups[viewer.gi].items.length - 1}
                onPress={() => setViewer({ gi: viewer.gi, k: viewer.k + 1 })}
              ><Text style={s.navTxt}>›</Text></TouchableOpacity>
            </View>
          </View>
        )}
      </Modal>

      {/* Revisión */}
      <Modal visible={review} animationType="slide" transparent onRequestClose={() => setReview(false)}>
        <View style={s.sheetBg}>
          <View style={s.sheet}>
            <Text style={s.h2}>Revisar cambios</Text>
            <Text style={s.lead}>{toDelete.length} para borrar · {toMove.length} para mover</Text>
            <ScrollView style={{ maxHeight: 260 }}>
              {toDelete.concat(toMove).map((a) => (
                <View key={a.id} style={s.item}>
                  <Image source={{ uri: a.uri }} style={s.itemImg} />
                  <View style={{ flex: 1 }}>
                    <Text numberOfLines={1} style={s.name}>{a.filename}</Text>
                    <Text style={s.meta}>{fmtSize(a.size)}</Text>
                  </View>
                  <Text style={[s.pill, { backgroundColor: st(a.id) === 'delete' ? C.del : C.move }]}>{LABEL[st(a.id)]}</Text>
                </View>
              ))}
            </ScrollView>
            {toMove.length > 0 && (
              <View style={{ marginTop: 12 }}>
                <Text style={s.name}>Mover a la carpeta</Text>
                <TextInput style={s.input} value={dest} onChangeText={setDest} />
              </View>
            )}
            <Text style={s.small}>Android te pedirá confirmar antes de borrar o mover.</Text>
            <View style={{ height: 8 }} />
            <Btn title="Aplicar cambios" kind="primary" onPress={apply} />
            <View style={{ height: 8 }} />
            <Btn title="Volver" onPress={() => setReview(false)} />
          </View>
        </View>
      </Modal>
    </View>
  );
}

const s = StyleSheet.create({
  root: { flex: 1, backgroundColor: C.bg },
  pad: { padding: 16 },
  h1: { fontSize: 28, fontWeight: '800', color: C.ink, marginBottom: 6 },
  h2: { fontSize: 20, fontWeight: '800', color: C.ink },
  lead: { fontSize: 16, color: C.muted, marginBottom: 16, lineHeight: 22 },
  small: { fontSize: 13, color: C.muted, marginTop: 14 },
  btn: { minHeight: 48, borderRadius: 12, borderWidth: 1, borderColor: C.line, backgroundColor: C.card, alignItems: 'center', justifyContent: 'center', paddingHorizontal: 16 },
  btnPrimary: { backgroundColor: C.accent, borderColor: C.accent },
  btnTxt: { fontWeight: '700', fontSize: 15, color: C.ink, textAlign: 'center' },
  row: { backgroundColor: C.card, borderWidth: 1, borderColor: C.line, borderRadius: 14, padding: 14, marginBottom: 8 },
  rowTitle: { fontSize: 16, fontWeight: '700', color: C.ink },
  rowSub: { fontSize: 13, color: C.muted, marginTop: 2 },
  bar: { height: 10, borderRadius: 5, backgroundColor: C.line, overflow: 'hidden', marginBottom: 12 },
  barFill: { height: 10, backgroundColor: C.accent },
  stats: { flexDirection: 'row', gap: 8, marginBottom: 14 },
  stat: { flex: 1, backgroundColor: C.card, borderWidth: 1, borderColor: C.line, borderRadius: 14, padding: 10 },
  statN: { fontSize: 20, fontWeight: '800', color: C.ink },
  statL: { fontSize: 12, color: C.muted },
  group: { backgroundColor: C.card, borderWidth: 1, borderColor: C.line, borderRadius: 16, padding: 12, marginTop: 14 },
  groupTitle: { fontWeight: '800', color: C.ink, marginBottom: 10 },
  tiles: { flexDirection: 'row', flexWrap: 'wrap', gap: 10 },
  thumb: { width: '100%', height: TILE * 0.75, borderRadius: 10, backgroundColor: C.line },
  vid: { position: 'absolute', top: 6, left: 6, backgroundColor: 'rgba(0,0,0,.65)', borderRadius: 99, paddingHorizontal: 8, paddingVertical: 2 },
  vidTxt: { color: '#fff', fontSize: 11, fontWeight: '700' },
  badge: { position: 'absolute', left: 0, right: 0, bottom: 0, paddingHorizontal: 8, paddingVertical: 3, borderBottomLeftRadius: 10, borderBottomRightRadius: 10 },
  badgeTxt: { color: '#fff', fontSize: 12, fontWeight: '700' },
  name: { fontSize: 13, fontWeight: '700', color: C.ink, marginTop: 6 },
  meta: { fontSize: 12, color: C.muted, marginBottom: 6 },
  seg: { flexDirection: 'row', gap: 4 },
  segBtn: { flex: 1, height: 38, borderWidth: 1, borderColor: C.line, borderRadius: 9, alignItems: 'center', justifyContent: 'center' },
  segTxt: { fontSize: 11, fontWeight: '700', color: C.muted },
  dock: { position: 'absolute', left: 0, right: 0, bottom: 0, backgroundColor: C.card, borderTopWidth: 1, borderColor: C.line, padding: 12, flexDirection: 'row', alignItems: 'center', justifyContent: 'space-between', gap: 12 },
  dockTxt: { flex: 1, fontSize: 14, color: C.ink, fontWeight: '600' },
  viewer: { flex: 1, backgroundColor: '#0b0f13', paddingTop: 30 },
  vTop: { flexDirection: 'row', justifyContent: 'space-between', alignItems: 'center', padding: 12 },
  close: { color: '#fff', fontWeight: '700', padding: 8 },
  vImg: { flex: 1, width: '100%' },
  vInfo: { color: '#c7d0d9', textAlign: 'center', fontSize: 13, padding: 12 },
  vBar: { flexDirection: 'row', alignItems: 'center', gap: 8, padding: 12, paddingBottom: 24 },
  nav: { width: 48, height: 48, borderRadius: 12, backgroundColor: 'rgba(255,255,255,.14)', alignItems: 'center', justifyContent: 'center' },
  navTxt: { color: '#fff', fontSize: 26 },
  sheetBg: { flex: 1, backgroundColor: 'rgba(8,12,16,.55)', justifyContent: 'flex-end' },
  sheet: { backgroundColor: C.card, borderTopLeftRadius: 20, borderTopRightRadius: 20, padding: 16, paddingBottom: 24 },
  item: { flexDirection: 'row', alignItems: 'center', gap: 10, marginBottom: 8 },
  itemImg: { width: 48, height: 48, borderRadius: 8, backgroundColor: C.line },
  pill: { color: '#fff', fontSize: 12, fontWeight: '700', paddingHorizontal: 9, paddingVertical: 3, borderRadius: 99, overflow: 'hidden' },
  input: { borderWidth: 1, borderColor: C.line, borderRadius: 10, height: 46, paddingHorizontal: 12, marginTop: 6, color: C.ink, fontSize: 16 },
});
