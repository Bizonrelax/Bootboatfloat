# ARCHIVE 02 — BootboatClaims v1.1.0
Назначение: приват от 15000, бюджет 1 км² суммарно, зазор 100 м, БУВ +100 м без
взрывов, лимиты 100 приватов и сторона 10…10000 (решение владельца 19.11.2026,
расчёт K_max = 1 210 000/(s_min+100)²), /claim square и /claim budget, fail-closed.
Заменяет версию 1.0.0 от 18.11 («без лимитов») — ту кодеру не давать.
Стек: Paper 26.3, Java 25, Maven. Компиляция только против paper-api.
GriefPrevention подключается рефлексией, при сборке не нужен.
Репозиторий: Bizonrelax/Bootboatfloat. Документ: archive/g/ARCHIVE-02-Claims.md.
Порядок серии: 01 Spawn, 02 Claims (этот), 03 Archive, 04 Home (последним).
Инструкция разбирающему: создать файлы точно по путям ниже, содержимое между
маркерами FILE брать целиком, маркеры в файлы не включать.

=== FILE: g/BootboatClaims/pom.xml ===
<project xmlns="http://maven.apache.org/POM/4.0.0"
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
<modelVersion>4.0.0</modelVersion>
<groupId>ru.bootboat</groupId>
<artifactId>bootboat-claims</artifactId>
<version>1.1.0</version>
<packaging>jar</packaging>
<properties>
<maven.compiler.release>25</maven.compiler.release>
<project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
</properties>
<repositories>
<repository>
<id>papermc</id>
<url>https://repo.papermc.io/repository/maven-public/</url>
</repository>
</repositories>
<dependencies>
<dependency>
<groupId>io.papermc.paper</groupId>
<artifactId>paper-api</artifactId>
<version>26.3-R0.1-SNAPSHOT</version>
<scope>provided</scope>
</dependency>
<dependency>
<groupId>org.junit.jupiter</groupId>
<artifactId>junit-jupiter</artifactId>
<version>5.10.2</version>
<scope>test</scope>
</dependency>
</dependencies>
<build>
<plugins>
<plugin>
<groupId>org.apache.maven.plugins</groupId>
<artifactId>maven-surefire-plugin</artifactId>
<version>3.2.5</version>
</plugin>
</plugins>
</build>
</project>

=== FILE: g/BootboatClaims/src/main/resources/plugin.yml ===
name: BootboatClaims
version: 1.1.0
main: ru.bootboat.claims.BootboatClaims
api-version: '26.3'
softdepend: [GriefPrevention]
commands:
claim:
description: Приват в пределах бюджета и лимитов
usage: /claim square <сторона> | /claim budget
permissions:
bootboat.claims.use:
default: true

=== FILE: g/BootboatClaims/src/main/resources/config.yml ===
world: world
min-distance: 15000          # приват целиком вне квадрата +-15000
budget-area: 1000000         # 1 км² суммарно на игрока
min-side: 10                 # решение 19.11: минимум стороны
max-side: 10000              # решение 19.11: максимум стороны (длинная граница до 10 км)
min-area: 100                # 10x10
max-claims-per-player: 100   # решение 19.11: K_max = 1210000/(min-side+100)^2
claim-gap: 100               # зазор между любыми приватами, включая свои
buffer-pad: 100              # БУВ: приват + 100 м по всем сторонам
buffer-no-explosions: true
buffer-protect-entities: true
buffer-counts-against-budget: false
claim-min-interval-ms: 1000  # антифриз: не чаще одного привата в секунду
fail-closed: true
block-commands:
- restorenature
- restorenatureaggressive
- deleteclaimsinworld
- deleteallclaims
messages:
no-perm: "Нет права."
player-only: "Только для игрока."
usage: "Приват: /claim square <сторона> или /claim budget. Лопата GP тоже работает."
budget: "Приват: занято {used} м² из {budget} м², остаток {left} м², зон {count} из {maxcount}. Порог: не ближе {min} от нуля."
too-close: "Приват должен целиком лежать вне квадрата +- {min} от нуля."
too-small: "Слишком мало: минимум {min} м²."
too-small-side: "Сторона меньше минимума {min} м."
too-big-side: "Сторона больше лимита {max} м."
denied-count: "Лимит зон: {max} на игрока. Сними старую или расширяй существующую."
no-budget: "Не хватает бюджета: нужно {need} м², остаток {left} м². Максимальная сторона сейчас: {maxside}."
created: "Приват создан: {minx}..{maxx} x {minz}..{maxz}, площадь {area} м². Остаток бюджета {left} м²."
create-fail: "Приват не создан: {reason}. Можно поставить золотой лопатой GP."
denied-distance: "Приват отклонён: он заходит в квадрат +- {min} от нуля."
denied-budget: "Приват отклонён: превышен бюджет {budget} м² (вышло бы {total} м²)."
denied-area: "Приват отклонён: площадь меньше {min} м²."
denied-gap: "Приват отклонён: до другого привата меньше {gap} м. Даже до своего."
too-fast: "Не чаще одного привата в {ms} мс — иначе сервер встанет. Бюджет твой не тронут."
cmd-blocked: "Команда отключена: массовое восстановление природы вешает сервер."
gp-missing: "Приваты временно закрыты: GriefPrevention не отвечает. Сообщи владельцу."
wrong-world: "Приватиться можно только в мире {world}."

=== FILE: g/BootboatClaims/src/main/java/ru/bootboat/claims/GpClaims.java ===
package ru.bootboat.claims;
import org.bukkit.Bukkit;
import org.bukkit.Location;
import org.bukkit.World;
import org.bukkit.entity.Player;
import org.bukkit.plugin.Plugin;
import java.lang.reflect.Field;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
import java.util.UUID;
import java.util.logging.Logger;
/**
* Адаптер GriefPrevention через рефлексию. Компилируется без GP.
* Для BootboatHome и BootboatArchive: копия файла, заменить package.
*/
public final class GpClaims {
public static final class Rect {
public final UUID owner;      // null = админский приват
public final String world;
public final int minX, minZ, maxX, maxZ;
Rect(UUID owner, String world, int minX, int minZ, int maxX, int maxZ) {
this.owner = owner;
this.world = world;
this.minX = minX;
this.minZ = minZ;
this.maxX = maxX;
this.maxZ = maxZ;
}
public int area() {
return (maxX - minX + 1) * (maxZ - minZ + 1);
}
public boolean same(Rect o) {
return o != null && world.equals(o.world)
&& minX == o.minX && minZ == o.minZ && maxX == o.maxX && maxZ == o.maxZ;
}
public boolean intersectsSquare(String w, int min, int max) {
return world.equals(w) && minX <= max && maxX >= min && minZ <= max && maxZ >= min;
}
}
private final Logger log;
private Object dataStore;
private Class<?> claimClass;
private Method mGetClaimAt, mGetPlayerData, mOwner, mLesser, mGreater;
private Field fOwner, fLesser, fGreater, fParent;
private boolean ready;
public GpClaims(Logger log) {
this.log = log;
init();
}
private void init() {
try {
Plugin gp = Bukkit.getPluginManager().getPlugin("GriefPrevention");
if (gp == null) {
log.warning("GriefPrevention не найден — проверки приватов не работают.");
return;
}
claimClass = Class.forName("me.ryanhamshire.GriefPrevention.Claim");
Field ds = gp.getClass().getField("dataStore");
dataStore = ds.get(gp);
if (dataStore == null) {
log.warning("GP dataStore пуст — проверки приватов не работают.");
return;
}
mGetClaimAt = dataStore.getClass().getMethod("getClaimAt", Location.class, boolean.class, claimClass);
mGetPlayerData = dataStore.getClass().getMethod("getPlayerData", UUID.class);
mOwner = method(claimClass, "getOwnerID");
mLesser = method(claimClass, "getLesserBoundaryCorner");
mGreater = method(claimClass, "getGreaterBoundaryCorner");
fOwner = field(claimClass, "ownerID");
fLesser = field(claimClass, "lesserBoundaryCorner");
fGreater = field(claimClass, "greaterBoundaryCorner");
fParent = field(claimClass, "parent");
if (mGetClaimAt == null || (mOwner == null && fOwner == null)
|| (mLesser == null && fLesser == null) || (mGreater == null && fGreater == null)) {
log.severe("GP API не распознан (Claim/DataStore). Проверки приватов НЕ работают.");
return;
}
ready = true;
log.info("GriefPrevention подключён через рефлексию.");
} catch (Throwable t) {
log.severe("GP API не распознан: " + t + ". Проверки приватов НЕ работают.");
}
}
private static Method method(Class<?> c, String name) {
try {
Method m = c.getMethod(name);
m.setAccessible(true);
return m;
} catch (Throwable t) {
return null;
}
}
private static Field field(Class<?> c, String name) {
try {
Field f = c.getField(name);
f.setAccessible(true);
return f;
} catch (Throwable t) {
return null;
}
}
public boolean ready() {
return ready;
}
public Rect rectOfClaim(Object claim) {
if (claim == null) return null;
try {
UUID owner = (UUID) (mOwner != null ? mOwner.invoke(claim) : fOwner.get(claim));
Location a = (Location) (mLesser != null ? mLesser.invoke(claim) : fLesser.get(claim));
Location b = (Location) (mGreater != null ? mGreater.invoke(claim) : fGreater.get(claim));
if (a == null || b == null) return null;
String w = a.getWorld() == null ? "" : a.getWorld().getName();
return new Rect(owner, w,
Math.min(a.getBlockX(), b.getBlockX()), Math.min(a.getBlockZ(), b.getBlockZ()),
Math.max(a.getBlockX(), b.getBlockX()), Math.max(a.getBlockZ(), b.getBlockZ()));
} catch (Throwable t) {
return null;
}
}
/** null = привата нет. Rect с owner == null = админский приват. */
public Rect rectAt(Location loc) {
if (!ready) return null;
try {
Object claim = mGetClaimAt.invoke(dataStore, loc, true, null);
return rectOfClaim(claim);
} catch (Throwable t) {
return null;
}
}
public boolean isForeign(Location loc, UUID player) {
Rect r = rectAt(loc);
return r != null && (r.owner == null || !r.owner.equals(player));
}
public List<Rect> claimsOf(UUID player) {
List<Rect> out = new ArrayList<>();
if (!ready || mGetPlayerData == null) return out;
try {
Object pd = mGetPlayerData.invoke(dataStore, player);
if (pd == null) return out;
Method gc = pd.getClass().getMethod("getClaims");
Object list = gc.invoke(pd);
if (!(list instanceof List<?> l)) return out;
for (Object c : l) {
if (c == null) continue;
if (fParent != null && fParent.get(c) != null) continue; // подразделы не считаем
Rect r = rectOfClaim(c);
if (r != null) out.add(r);
}
} catch (Throwable t) {
log.warning("GP claimsOf: " + t);
}
return out;
}
/** Все приваты сервера, включая админские. Для индекса и для Archive. */
public List<Rect> allClaims() {
List<Rect> out = new ArrayList<>();
if (!ready) return out;
try {
Method m = dataStore.getClass().getMethod("getClaims");
Object list = m.invoke(dataStore);
if (list instanceof List<?> l) {
for (Object c : l) {
Rect r = rectOfClaim(c);
if (r != null) out.add(r);
}
}
} catch (Throwable t) {
log.warning("GP allClaims: " + t);
}
return out;
}
/** Создание привата без золотой лопаты. null = успех, иначе текст ошибки. */
public String createClaim(World w, int x1, int x2, int y1, int y2, int z1, int z2, UUID owner, Player p) {
if (!ready) return "GP API недоступен";
try {
Method target = null;
for (Method m : dataStore.getClass().getMethods()) {
if (!m.getName().equals("createClaim")) continue;
Class<?>[] pt = m.getParameterTypes();
int ints = 0;
boolean hasWorld = false, hasUuid = false;
for (Class<?> c : pt) {
if (c == int.class) ints++;
if (c == World.class) hasWorld = true;
if (c == UUID.class) hasUuid = true;
}
if (ints == 6 && hasWorld && hasUuid) {
target = m;
break;
}
}
if (target == null) return "метод createClaim не найден в этой сборке GP";
Class<?>[] pt = target.getParameterTypes();
Object[] args = new Object[pt.length];
int[] vals = {x1, x2, y1, y2, z1, z2};
int ii = 0;
for (int i = 0; i < pt.length; i++) {
Class<?> c = pt[i];
if (c == World.class) args[i] = w;
else if (c == int.class) args[i] = vals[ii++];
else if (c == UUID.class) args[i] = owner;
else if (c == Player.class) args[i] = p;
else if (c == boolean.class) args[i] = false;
else if (c == long.class) args[i] = 0L;
else args[i] = null;
}
Object res = target.invoke(dataStore, args);
if (res == null) return "createClaim вернул null";
Boolean ok = readBool(res, "succeeded", "success");
if (ok != null && !ok) return "GP отказал: пересечение с чужим приватом или не хватает блоков привата";
return null;
} catch (Throwable t) {
return "createClaim: " + t;
}
}
private static Boolean readBool(Object o, String... names) {
for (String n : names) {
try {
Field f = o.getClass().getField(n);
if (f.getType() == boolean.class) return f.getBoolean(o);
} catch (Throwable ignored) {
}
try {
Method m = o.getClass().getMethod(n);
if (m.getReturnType() == boolean.class) return (Boolean) m.invoke(o);
} catch (Throwable ignored) {
}
}
return null;
}
}

=== FILE: g/BootboatClaims/src/main/java/ru/bootboat/claims/ClaimSpatial.java ===
package ru.bootboat.claims;
import java.util.List;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.CopyOnWriteArrayList;
/**
* Пространственный индекс приватов: корзины по регионам 512×512.
* Зазор и буфер — один и тот же запрос прямоугольником.
* Правило 100 м само держит плотность ~25 приватов на регион, поэтому запрос константный.
*/
public final class ClaimSpatial {
private final ConcurrentHashMap<Long, CopyOnWriteArrayList<GpClaims.Rect>> buckets = new ConcurrentHashMap<>();
private volatile int count;
private static long key(int rx, int rz) {
return ((long) rx << 32) | (rz & 0xffffffffL);
}
public void add(GpClaims.Rect r) {
for (int rx = r.minX >> 9; rx <= r.maxX >> 9; rx++) {
for (int rz = r.minZ >> 9; rz <= r.maxZ >> 9; rz++) {
buckets.computeIfAbsent(key(rx, rz), k -> new CopyOnWriteArrayList<>()).add(r);
}
}
count++;
}
public void remove(GpClaims.Rect r) {
boolean hit = false;
for (int rx = r.minX >> 9; rx <= r.maxX >> 9; rx++) {
for (int rz = r.minZ >> 9; rz <= r.maxZ >> 9; rz++) {
var b = buckets.get(key(rx, rz));
if (b == null) continue;
for (GpClaims.Rect x : b) {
if (x.same(r)) {
b.remove(x);
hit = true;
break;
}
}
if (b.isEmpty()) buckets.remove(key(rx, rz));
}
}
if (hit && count > 0) count--;
}
public void clear() {
buckets.clear();
count = 0;
}
/** Есть ли приват, пересекающий прямоугольник запроса. exclude — сам приват при resize. */
public boolean intersects(String world, int minX, int minZ, int maxX, int maxZ, GpClaims.Rect... exclude) {
for (int rx = minX >> 9; rx <= maxX >> 9; rx++) {
for (int rz = minZ >> 9; rz <= maxZ >> 9; rz++) {
List<GpClaims.Rect> list = buckets.get(key(rx, rz));
if (list == null) continue;
for (GpClaims.Rect r : list) {
if (!world.equals(r.world)) continue;
if (r.minX > maxX || r.maxX < minX || r.minZ > maxZ || r.maxZ < minZ) continue;
boolean skip = false;
for (GpClaims.Rect ex : exclude) if (ex != null && ex.same(r)) skip = true;
if (!skip) return true;
}
}
}
return false;
}
/** Точка в БУВ: приват в пределах pad по Чебышёву. */
public boolean bufferContains(String world, int x, int z, int pad) {
return intersects(world, x - pad, z - pad, x + pad, z + pad);
}
public int size() {
return count;
}
}

=== FILE: g/BootboatClaims/src/main/java/ru/bootboat/claims/BootboatClaims.java ===
package ru.bootboat.claims;
import org.bukkit.Bukkit;
import org.bukkit.Material;
import org.bukkit.World;
import org.bukkit.command.Command;
import org.bukkit.command.CommandSender;
import org.bukkit.command.TabExecutor;
import org.bukkit.entity.Player;
import org.bukkit.event.Cancellable;
import org.bukkit.event.Event;
import org.bukkit.event.EventPriority;
import org.bukkit.event.Listener;
import org.bukkit.event.block.BlockExplodeEvent;
import org.bukkit.event.entity.EntityDamageByEntityEvent;
import org.bukkit.event.entity.EntityDamageEvent;
import org.bukkit.event.entity.EntityExplodeEvent;
import org.bukkit.event.player.PlayerCommandPreprocessEvent;
import org.bukkit.event.player.PlayerInteractEvent;
import org.bukkit.plugin.java.JavaPlugin;
import java.lang.reflect.Method;
import java.util.ArrayList;
import java.util.List;
import java.util.Locale;
import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;
public final class BootboatClaims extends JavaPlugin implements Listener, TabExecutor {
private GpClaims gp;
private String worldName;
private int minDistance, budget, minArea, minSide, maxSide, maxClaims;
private boolean failClosed;
private List<String> blockedCommands;
private int claimGap, bufferPad;
private boolean bufferNoExplosions, bufferProtectEntities, bufferInBudget;
private ClaimSpatial spatial;
private int claimIntervalMs;
private final Map<String, Long> lastClaim = new ConcurrentHashMap<>();
@Override
public void onEnable() {
saveDefaultConfig();
var cfg = getConfig();
worldName = cfg.getString("world", "world");
minDistance = cfg.getInt("min-distance", 15000);
budget = cfg.getInt("budget-area", 1000000);
minSide = Math.max(1, cfg.getInt("min-side", 10));
maxSide = cfg.getInt("max-side", 10000);
minArea = cfg.getInt("min-area", 100);
maxClaims = Math.max(0, cfg.getInt("max-claims-per-player", 100));
failClosed = cfg.getBoolean("fail-closed", true);
blockedCommands = cfg.getStringList("block-commands");
claimGap = Math.max(0, cfg.getInt("claim-gap", 100));
bufferPad = Math.max(0, cfg.getInt("buffer-pad", 100));
bufferNoExplosions = cfg.getBoolean("buffer-no-explosions", true);
bufferProtectEntities = cfg.getBoolean("buffer-protect-entities", true);
bufferInBudget = cfg.getBoolean("buffer-counts-against-budget", false);
claimIntervalMs = Math.max(0, cfg.getInt("claim-min-interval-ms", 1000));
gp = new GpClaims(getLogger());
spatial = new ClaimSpatial();
Bukkit.getPluginManager().registerEvents(this, this);
var cmd = getCommand("claim");
if (cmd != null) cmd.setExecutor(this);
hookClaimEvent("ClaimCreatedEvent", false);
hookClaimEvent("ClaimModifiedEvent", true);
rebuildSpatial();
hookDeleted();
getLogger().info("BootboatClaims: мир=" + worldName + ", порог=" + minDistance
+ ", бюджет=" + budget + " м², сторона=" + minSide + ".." + maxSide
+ ", зон=" + maxClaims + ", зазор=" + claimGap + ", БУВ=" + bufferPad);
if (!gp.ready()) {
getLogger().severe("GriefPrevention не распознан. fail-closed=" + failClosed
+ " (лопата будет заблокирована, чтобы не появилось непроверенных приватов).");
}
}
// ---------- индекс ----------
private void rebuildSpatial() {
spatial.clear();
if (!gp.ready()) {
getLogger().severe("GP недоступен: индекс пуст, зазор и БУВ НЕ защищены.");
return;
}
for (GpClaims.Rect r : gp.allClaims()) {
if (worldName.equals(r.world)) spatial.add(r);
}
getLogger().info("Индекс приватов: " + spatial.size() + ", зазор " + claimGap + ", БУВ " + bufferPad);
}
private boolean tooFast(String nick) {
if (claimIntervalMs <= 0) return false;
long now = System.currentTimeMillis();
Long prev = lastClaim.put(nick.toLowerCase(Locale.ROOT), now);
return prev != null && now - prev < claimIntervalMs;
}
@SuppressWarnings("unchecked")
private void hookDeleted() {
try {
Class<? extends Event> clazz = (Class<? extends Event>)
Class.forName("me.ryanhamshire.GriefPrevention.events.ClaimDeletedEvent");
Bukkit.getPluginManager().registerEvent(clazz, this, EventPriority.MONITOR, (l, e) -> {
GpClaims.Rect r = gp.rectOfClaim(extract(e, new String[]{"getClaim"}));
if (r != null) spatial.remove(r);
}, this, true);
} catch (ClassNotFoundException e) {
getLogger().warning("ClaimDeletedEvent не найден — индекс будет расходиться после удаления приватов.");
}
}
// ---------- перехват событий GP ----------
@SuppressWarnings("unchecked")
private void hookClaimEvent(String simple, boolean modified) {
try {
Class<? extends Event> clazz = (Class<? extends Event>)
Class.forName("me.ryanhamshire.GriefPrevention.events." + simple);
Bukkit.getPluginManager().registerEvent(clazz, this, EventPriority.LOWEST, (listener, event) -> {
if (!clazz.isInstance(event)) return;
try {
Object to = extract(event, modified
? new String[]{"getTo", "getNewClaim", "getClaim"}
: new String[]{"getClaim"});
GpClaims.Rect rect = gp.rectOfClaim(to);
if (rect == null || !worldName.equals(rect.world)) return;
GpClaims.Rect from = null;
if (modified) {
Object f = extract(event, new String[]{"getFrom", "getOldClaim"});
from = gp.rectOfClaim(f);
}
String err = validate(rect, from);
if (err != null) {
if (event instanceof Cancellable c) c.setCancelled(true);
if (rect.owner != null) {
Player p = Bukkit.getPlayer(rect.owner);
if (p != null) p.sendMessage(err);
}
getLogger().info("Приват отклонён (" + simple + "): " + err);
} else {
if (from != null) spatial.remove(from);
spatial.add(rect);
}
} catch (Throwable t) {
getLogger().warning("GP event " + simple + ": " + t);
}
}, this, true);
getLogger().info("Перехвачено событие GP: " + simple);
} catch (ClassNotFoundException e) {
getLogger().warning("Событие GP не найдено: " + simple + " — этот путь привата не проверяется.");
}
}
private static Object extract(Object event, String[] names) {
for (String n : names) {
try {
Method m = event.getClass().getMethod(n);
Object v = m.invoke(event);
if (v != null) return v;
} catch (Throwable ignored) {
}
}
return null;
}
private static GpClaims.Rect expand(GpClaims.Rect r, int d) {
return new GpClaims.Rect(r.owner, r.world, r.minX - d, r.minZ - d, r.maxX + d, r.maxZ + d);
}
private int footprint(GpClaims.Rect r) {
if (!bufferInBudget) return r.area();
long w = r.maxX - r.minX + 1L + 2L * bufferPad;
long h = r.maxZ - r.minZ + 1L + 2L * bufferPad;
long a = w * h;
return a > Integer.MAX_VALUE ? Integer.MAX_VALUE : (int) a;
}
/** null = разрешено, иначе текст отказа. Лимиты 19.11: сторона и число зон. */
private String validate(GpClaims.Rect to, GpClaims.Rect from) {
if (to.intersectsSquare(worldName, -minDistance, minDistance)) {
return text("denied-distance", "min", str(minDistance));
}
if (to.area() < minArea) {
return text("denied-area", "min", str(minArea));
}
int w = to.maxX - to.minX + 1, h = to.maxZ - to.minZ + 1;
if (w < minSide || h < minSide) {
return text("too-small-side", "min", str(minSide));
}
if (maxSide > 0 && (w > maxSide || h > maxSide)) {
return text("too-big-side", "max", str(maxSide));
}
if (claimGap > 0) {
GpClaims.Rect probe = expand(to, claimGap - 1);   // -1: зазор ровно 100 разрешён
if (spatial.intersects(worldName, probe.minX, probe.minZ, probe.maxX, probe.maxZ, to, from)) {
return text("denied-gap", "gap", str(claimGap));
}
}
if (to.owner == null) return null; // админский приват не ограничиваем
List<GpClaims.Rect> list = own(to.owner);
if (from == null && maxClaims > 0 && list.size() >= maxClaims) {
return text("denied-count", "max", str(maxClaims));
}
int used = 0;
boolean toCounted = false, fromCounted = false;
for (GpClaims.Rect r : list) {
used += footprint(r);
if (r.same(to)) toCounted = true;
if (from != null && r.same(from)) fromCounted = true;
}
int total = used + (toCounted ? 0 : footprint(to)) - (fromCounted ? footprint(from) : 0);
if (total > budget) {
return text("denied-budget", "budget", str(budget), "total", str(total));
}
return null;
}
private List<GpClaims.Rect> own(UUID id) {
List<GpClaims.Rect> out = new ArrayList<>();
for (GpClaims.Rect r : gp.claimsOf(id)) {
if (worldName.equals(r.world)) out.add(r);
}
return out;
}
private int used(UUID id) {
int sum = 0;
for (GpClaims.Rect r : own(id)) sum += r.area();
return sum;
}
// ---------- команды ----------
@Override
public boolean onCommand(CommandSender sender, Command cmd, String label, String[] args) {
if (!(sender instanceof Player p)) {
msg(sender, "player-only");
return true;
}
if (!p.hasPermission("bootboat.claims.use")) {
msg(p, "no-perm");
return true;
}
if (args.length == 0) {
msg(p, "usage");
return true;
}
if (args[0].equalsIgnoreCase("budget")) {
int used = used(p.getUniqueId());
msg(p, "budget", "used", str(used), "budget", str(budget),
"left", str(Math.max(0, budget - used)), "count", str(own(p.getUniqueId()).size()),
"maxcount", str(maxClaims), "min", str(minDistance));
return true;
}
if (args[0].equalsIgnoreCase("square")) {
if (args.length < 2) {
p.sendMessage("/claim square <сторона>");
return true;
}
int side;
try {
side = Integer.parseInt(args[1]);
} catch (NumberFormatException e) {
p.sendMessage("/claim square <сторона>");
return true;
}
square(p, side);
return true;
}
msg(p, "usage");
return true;
}
@Override
public List<String> onTabComplete(CommandSender sender, Command cmd, String alias, String[] args) {
List<String> out = new ArrayList<>();
if (args.length == 1) {
out.add("square");
out.add("budget");
}
return out;
}
private void square(Player p, int side) {
if (tooFast(p.getName())) {
p.sendMessage(text("too-fast", "ms", str(claimIntervalMs)));
return;
}
if (!gp.ready()) {
msg(p, "gp-missing");
return;
}
World w = p.getWorld();
if (!worldName.equals(w.getName())) {
msg(p, "wrong-world", "world", worldName);
return;
}
if (side < minSide) {
msg(p, "too-small-side", "min", str(minSide));
return;
}
if (maxSide > 0 && side > maxSide) {
msg(p, "too-big-side", "max", str(maxSide));
return;
}
if (maxClaims > 0 && own(p.getUniqueId()).size() >= maxClaims) {
msg(p, "denied-count", "max", str(maxClaims));
return;
}
int left = Math.max(0, budget - used(p.getUniqueId()));
if (side * side > left) {
msg(p, "no-budget", "need", str(side * side), "left", str(left),
"maxside", str((int) Math.floor(Math.sqrt(left))));
return;
}
int px = p.getLocation().getBlockX(), pz = p.getLocation().getBlockZ();
int minX = px - side / 2, minZ = pz - side / 2;
int maxX = minX + side - 1, maxZ = minZ + side - 1;
GpClaims.Rect rect = new GpClaims.Rect(p.getUniqueId(), worldName, minX, minZ, maxX, maxZ);
String err = validate(rect, null);
if (err != null) {
p.sendMessage(err);
return;
}
String fail = gp.createClaim(w, minX, maxX, w.getMinHeight(), w.getMaxHeight() - 1,
minZ, maxZ, p.getUniqueId(), p);
if (fail != null) {
msg(p, "create-fail", "reason", fail);
return;
}
int nowLeft = Math.max(0, budget - used(p.getUniqueId()));
msg(p, "created", "minx", str(minX), "maxx", str(maxX), "minz", str(minZ), "maxz", str(maxZ),
"area", str(side * side), "left", str(nowLeft));
}
// ---------- закрытие зависаний, fail-closed, скорость ----------
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onCommand(PlayerCommandPreprocessEvent e) {
String msg = e.getMessage().toLowerCase(Locale.ROOT);
int sp = msg.indexOf(' ');
String head = (sp < 0 ? msg : msg.substring(0, sp)).substring(1);
int colon = head.indexOf(':');
if (colon >= 0) head = head.substring(colon + 1);
for (String bad : blockedCommands) {
if (head.equalsIgnoreCase(bad)) {
e.setCancelled(true);
msg(e.getPlayer(), "cmd-blocked");
getLogger().warning("Заблокирована команда от " + e.getPlayer().getName() + ": " + head);
return;
}
}
}
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onShovel(PlayerInteractEvent e) {
if (gp.ready() || !failClosed) return;
var item = e.getItem();
if (item == null || item.getType() != Material.GOLDEN_SHOVEL) return;
e.setCancelled(true);
msg(e.getPlayer(), "gp-missing");
}
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onShovelSpeed(PlayerInteractEvent e) {
var item = e.getItem();
if (item == null || item.getType() != Material.GOLDEN_SHOVEL) return;
if (e.getAction() != org.bukkit.event.block.Action.RIGHT_CLICK_BLOCK) return;
if (tooFast(e.getPlayer().getName())) {
e.setCancelled(true);
e.getPlayer().sendMessage(text("too-fast", "ms", str(claimIntervalMs)));
}
}
// ---------- БУВ: взрывы ----------
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onEntityExplode(EntityExplodeEvent e) {
if (!bufferNoExplosions) return;
var l = e.getLocation();
if (spatial.bufferContains(l.getWorld().getName(), l.getBlockX(), l.getBlockZ(), bufferPad)) {
e.setCancelled(true);
return;
}
e.blockList().removeIf(b -> spatial.bufferContains(b.getWorld().getName(), b.getX(), b.getZ(), bufferPad));
}
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onBlockExplode(BlockExplodeEvent e) {
if (!bufferNoExplosions) return;
var l = e.getBlock().getLocation();
if (spatial.bufferContains(l.getWorld().getName(), l.getBlockX(), l.getBlockZ(), bufferPad)) {
e.setCancelled(true);
return;
}
e.blockList().removeIf(b -> spatial.bufferContains(b.getWorld().getName(), b.getX(), b.getZ(), bufferPad));
}
@EventHandler(priority = EventPriority.LOWEST, ignoreCancelled = true)
public void onBlastDamage(EntityDamageByEntityEvent e) {
if (!bufferNoExplosions || !bufferProtectEntities) return;
var c = e.getDamageCause();
if (c != EntityDamageEvent.DamageCause.ENTITY_EXPLOSION
&& c != EntityDamageEvent.DamageCause.BLOCK_EXPLOSION) return;
var l = e.getEntity().getLocation();
if (spatial.bufferContains(l.getWorld().getName(), l.getBlockX(), l.getBlockZ(), bufferPad)) {
e.setCancelled(true);
}
}
// ---------- утилиты ----------
private static String str(int v) {
return Integer.toString(v);
}
private String text(String key, String... kv) {
String s = getConfig().getString("messages." + key, key);
for (int i = 0; i + 1 < kv.length; i += 2) s = s.replace("{" + kv[i] + "}", kv[i + 1]);
return s;
}
private void msg(CommandSender to, String key, String... kv) {
String s = text(key, kv);
if (!s.isEmpty()) to.sendMessage(s);
}
}

=== FILE: g/BootboatClaims/src/test/java/ru/bootboat/claims/ClaimSpatialTest.java ===
package ru.bootboat.claims;
import org.junit.jupiter.api.Test;
import java.util.UUID;
import static org.junit.jupiter.api.Assertions.*;
/** Чистый индекс: зазор, БУВ, исключение себя, удаление, корзины через регион. */
class ClaimSpatialTest {
private static GpClaims.Rect rect(int minX, int minZ, int maxX, int maxZ) {
return new GpClaims.Rect(UUID.randomUUID(), "world", minX, minZ, maxX, maxZ);
}
@Test
void intersectsBasic() {
ClaimSpatial s = new ClaimSpatial();
s.add(rect(0, 0, 99, 99));
assertTrue(s.intersects("world", 50, 50, 150, 150));
assertFalse(s.intersects("world", 100, 100, 200, 200));
assertEquals(1, s.size());
}
@Test
void excludeSelfOnResize() {
ClaimSpatial s = new ClaimSpatial();
GpClaims.Rect r = rect(0, 0, 99, 99);
s.add(r);
assertFalse(s.intersects("world", 0, 0, 99, 99, r));
assertTrue(s.intersects("world", 0, 0, 99, 99));
}
@Test
void bufferPadChebyshev() {
ClaimSpatial s = new ClaimSpatial();
s.add(rect(0, 0, 99, 99));
assertTrue(s.bufferContains("world", 199, 50, 100));
assertFalse(s.bufferContains("world", 200, 50, 100));
assertTrue(s.bufferContains("world", -100, -100, 100));
assertFalse(s.bufferContains("world", -101, 0, 100));
}
@Test
void removeClears() {
ClaimSpatial s = new ClaimSpatial();
GpClaims.Rect r = rect(0, 0, 99, 99);
s.add(r);
s.remove(r);
assertFalse(s.intersects("world", 0, 0, 99, 99));
assertEquals(0, s.size());
}
@Test
void spansRegionBuckets() {
ClaimSpatial s = new ClaimSpatial();
s.add(rect(500, 0, 600, 10));
assertTrue(s.intersects("world", 590, 5, 590, 5));
assertTrue(s.intersects("world", 505, 5, 505, 5));
}
@Test
void worldIsolation() {
ClaimSpatial s = new ClaimSpatial();
s.add(rect(0, 0, 99, 99));
assertFalse(s.intersects("other", 0, 0, 99, 99));
assertFalse(s.bufferContains("other", 50, 50, 100));
}
}

=== BUILD ===
cd g/BootboatClaims
mvn -B -ntp test        # ожидание: Tests run: 6, Failures: 0, Errors: 0
mvn -B -ntp package     # target/bootboat-claims-1.1.0.jar
Если paper-api 26.3-R0.1-SNAPSHOT не резолвится: открыть
https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/
взять точную строку версии, подставить в pom, изменение записать в отчёт.
Если Paper отвергнет api-version '26.3' — он напечатает допустимые значения в лог,
подставить их, изменение записать в отчёт. Больше ничего не менять.

=== INSTALL ===
jar в plugins/ сервера, перезапуск. config.yml создастся сам.
GriefPrevention обязан стоять: softdepend обеспечивает порядок загрузки.
Для одиночного теста min-distance можно снизить до 1000 — иначе бежать 15 км
ради одной команды. Перед открытием вернуть 15000.

=== PROMPT FOR CODER ===
Репозиторий: Bizonrelax/Bootboatfloat, ветка main.
Прочитай archive/g/STATE.md, затем archive/g/ARCHIVE-02-Claims.md.
Создай файлы ровно по путям из маркеров FILE, содержимое бери целиком.
Собери: mvn -B -ntp test, затем mvn -B -ntp package.
Ожидание: 6 тестов зелёные, jar bootboat-claims-1.1.0.jar.
Работай прямо в main, без веток и PR, минимум вызовов инструментов.
Пуш только после зелёного цикла. Jar приложи к отчёту файлом.
Отчёт: роль, шаги, число тестов, хеш коммита, вердикт ПУБЛИКОВАТЬ/НЕ ПУБЛИКОВАТЬ.
Если правил pom или api-version — напиши что и почему.
Ничего кроме этих файлов не создавать и не менять.

=== ACCEPTANCE ===
[ ] 6/6 юнит-тестов зелёные, skipped 0
[ ] jar собран и приложен к отчёту
[ ] в main один коммит, git status чистый
[ ] посторонние файлы не тронуты
Ручные проверки в игре (после установки GP):
C1 приват на 14999 отклонён; C2 на 15000 принят
C3 /claim square 1000 -> 1 000 000 м², бюджет исчерпан
C4 /claim square 100 после C3 -> отказ, остаток 0
C5 десять зон 300x300 с зазором 100 -> 900 000 м², остаток 100 000
C6 /claim square 10000 -> полоса невозможна квадратом (бюджет); лопатой 10000x100 разрешена
C7 /claim square 10001 -> отказ too-big-side; лопатой сторона 10001 -> отказ
C8 сторона 9 -> отказ too-small-side; 10x10 принят
C9 101-й приват -> отказ denied-count; после снятия одного — принят
C10 снятие своей зоны -> бюджет вернулся, природа не восстановилась
C11 /restorenature -> команда заблокирована
C12 GP выключен -> SEVERE в лог, лопата не работает
G1 два привата в 99 м -> второй отклонён; в 100 м -> принят (даже свои)
G2 крипер/ТНТ внутри БУВ -> блоки и сущности целы; вне БУВ -> взрыв обычный
G3 два привата подряд быстрее 1 с -> второй отклонён сообщением too-fast
