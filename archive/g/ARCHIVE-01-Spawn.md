# ARCHIVE 01 — BootboatSpawn v1.0.0
Назначение: случайный безопасный спавн новичка внутри зоны ±1500, один раз за всю игру.
Стек: Paper 26.3, Java 25, Maven. Компиляция только против paper-api.
AuthMe подключается рефлексией, при сборке не нужен.
Защита зоны — WorldGuard, НЕ этот плагин.

Инструкция разбирающему: создать файлы точно по путям ниже, содержимое between
маркерами FILE брать целиком, маркеры в файлы не включать.

=== FILE: Bootboatfloat/BootboatSpawn/pom.xml ===
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>ru.bootboat</groupId>
  <artifactId>bootboat-spawn</artifactId>
  <version>1.0.0</version>
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

=== FILE: Bootboatfloat/BootboatSpawn/src/main/resources/plugin.yml ===
name: BootboatSpawn
version: 1.0.0
main: ru.bootboat.spawn.BootboatSpawn
api-version: '26.3'
softdepend: [AuthMe]

=== FILE: Bootboatfloat/BootboatSpawn/src/main/resources/config.yml ===
world: world
radius: 1500
center-x: 0
center-z: 0
max-attempts: 40
message: "Ты появился в дикой зоне Bootboatfloat. Строить можно за пределами ±1500 от 0;0."

=== FILE: Bootboatfloat/BootboatSpawn/src/main/java/ru/bootboat/spawn/SpawnPicker.java ===
package ru.bootboat.spawn;

import java.util.Random;

/**
 * Чистая логика выбора точки спавна. Без Bukkit — тестируется на заглушке.
 * Возвращает null, если безопасной точки нет: вызывающий обязан использовать спавн мира.
 */
public final class SpawnPicker {

    public interface WorldView {
        int maxHeight();
        int highestY(int x, int z);
        boolean solid(int x, int y, int z);
        boolean passable(int x, int y, int z);
        boolean danger(int x, int y, int z);
    }

    public record Point(int x, int y, int z) {
    }

    private SpawnPicker() {
    }

    public static Point pick(WorldView w, Random rnd, int cx, int cz, int radius, int attempts) {
        int r = Math.max(0, radius);
        for (int i = 0; i < Math.max(1, attempts); i++) {
            int x = cx + rnd.nextInt(-r, r + 1);
            int z = cz + rnd.nextInt(-r, r + 1);
            int y = w.highestY(x, z);
            if (y + 2 >= w.maxHeight()) continue;
            if (!w.solid(x, y, z) || w.danger(x, y, z)) continue;
            if (!w.passable(x, y + 1, z) || !w.passable(x, y + 2, z)) continue;
            if (w.danger(x, y + 1, z) || w.danger(x, y + 2, z)) continue;
            return new Point(x, y + 1, z);
        }
        return null;
    }

    /** Первый вход — спавним. Маркер стоит — игрока не трогаем. */
    public static boolean shouldSpawn(boolean markerPresent) {
        return !markerPresent;
    }
}

=== FILE: Bootboatfloat/BootboatSpawn/src/main/java/ru/bootboat/spawn/BootboatSpawn.java ===
package ru.bootboat.spawn;

import org.bukkit.Bukkit;
import org.bukkit.HeightMap;
import org.bukkit.Location;
import org.bukkit.Material;
import org.bukkit.NamespacedKey;
import org.bukkit.World;
import org.bukkit.entity.Player;
import org.bukkit.event.Event;
import org.bukkit.event.EventHandler;
import org.bukkit.event.EventPriority;
import org.bukkit.event.Listener;
import org.bukkit.event.player.PlayerJoinEvent;
import org.bukkit.persistence.PersistentDataType;
import org.bukkit.plugin.java.JavaPlugin;

import java.util.Set;
import java.util.concurrent.ThreadLocalRandom;

public final class BootboatSpawn extends JavaPlugin implements Listener {

    private static final Set<Material> DANGER = Set.of(
            Material.LAVA, Material.WATER, Material.FIRE, Material.SOUL_FIRE,
            Material.MAGMA_BLOCK, Material.CACTUS, Material.SWEET_BERRY_BUSH,
            Material.WITHER_ROSE, Material.POWDER_SNOW, Material.CAMPFIRE,
            Material.SOUL_CAMPFIRE, Material.POINTED_DRIPSTONE, Material.BEDROCK);

    private NamespacedKey doneKey;
    private String worldName;
    private int radius, cx, cz, maxAttempts;
    private boolean authMe;

    @Override
    public void onEnable() {
        saveDefaultConfig();
        var cfg = getConfig();
        worldName = cfg.getString("world", "world");
        radius = Math.max(0, cfg.getInt("radius", 1500));
        cx = cfg.getInt("center-x", 0);
        cz = cfg.getInt("center-z", 0);
        maxAttempts = Math.max(1, cfg.getInt("max-attempts", 40));
        doneKey = new NamespacedKey(this, "spawn_done");
        authMe = Bukkit.getPluginManager().getPlugin("AuthMe") != null;
        Bukkit.getPluginManager().registerEvents(this, this);
        if (authMe) registerAuthMeListener();
        getLogger().info("BootboatSpawn: radius=" + radius + ", center=(" + cx + "," + cz
                + "), authMe=" + authMe + ", world=" + worldName);
    }

    private boolean hasMarker(Player p) {
        return p.getPersistentDataContainer().has(doneKey, PersistentDataType.BYTE);
    }

    @EventHandler(priority = EventPriority.MONITOR)
    public void onJoin(PlayerJoinEvent e) {
        Player p = e.getPlayer();
        if (!SpawnPicker.shouldSpawn(hasMarker(p))) return;
        if (!authMe) {
            Bukkit.getScheduler().runTaskLater(this, () -> spawnOnce(p), 1L);
        }
        // при AuthMe ждём LoginEvent: до /login игрока телепортировать нельзя
    }

    @SuppressWarnings("unchecked")
    private void registerAuthMeListener() {
        try {
            Class<? extends Event> clazz =
                    (Class<? extends Event>) Class.forName("fr.xephi.authme.events.LoginEvent");
            Bukkit.getPluginManager().registerEvent(clazz, this, EventPriority.MONITOR,
                    (listener, event) -> {
                        if (!clazz.isInstance(event)) return;
                        try {
                            Player p = (Player) clazz.getMethod("getPlayer").invoke(event);
                            Bukkit.getScheduler().runTaskLater(this, () -> spawnOnce(p), 1L);
                        } catch (ReflectiveOperationException ex) {
                            getLogger().warning("AuthMe reflection: " + ex);
                        }
                    }, this, true);
            getLogger().info("AuthMe LoginEvent перехвачен.");
        } catch (ClassNotFoundException ex) {
            getLogger().warning("AuthMe есть, но LoginEvent не найден — спавн на входе.");
            authMe = false;
        }
    }

    private void spawnOnce(Player p) {
        if (p == null || !p.isOnline()) return;
        var pdc = p.getPersistentDataContainer();
        if (!SpawnPicker.shouldSpawn(pdc.has(doneKey, PersistentDataType.BYTE))) return;
        World w = Bukkit.getWorld(worldName);
        if (w == null) {
            getLogger().warning("Мир " + worldName + " не найден");
            return;
        }
        Location loc = findSafe(w);
        if (loc == null) {
            loc = w.getSpawnLocation();
            getLogger().warning("Безопасная точка не найдена — спавн мира.");
        }
        final Location target = loc;
        p.teleportAsync(target).thenAccept(ok ->
                Bukkit.getScheduler().runTask(this, () -> {
                    if (Boolean.TRUE.equals(ok) && p.isOnline()) {
                        pdc.set(doneKey, PersistentDataType.BYTE, (byte) 1);
                        String msg = getConfig().getString("message", "");
                        if (msg != null && !msg.isEmpty()) p.sendMessage(msg);
                    }
                })
        );
    }

    private Location findSafe(World w) {
        SpawnPicker.WorldView view = new SpawnPicker.WorldView() {
            @Override
            public int maxHeight() {
                return w.getMaxHeight();
            }

            @Override
            public int highestY(int x, int z) {
                return w.getHighestBlockYAt(x, z, HeightMap.MOTION_BLOCKING_NO_LEAVES);
            }

            @Override
            public boolean solid(int x, int y, int z) {
                return w.getBlockAt(x, y, z).getType().isSolid();
            }

            @Override
            public boolean passable(int x, int y, int z) {
                return w.getBlockAt(x, y, z).isPassable();
            }

            @Override
            public boolean danger(int x, int y, int z) {
                return DANGER.contains(w.getBlockAt(x, y, z).getType());
            }
        };
        SpawnPicker.Point pt = SpawnPicker.pick(view, ThreadLocalRandom.current(), cx, cz, radius, maxAttempts);
        return pt == null ? null : new Location(w, pt.x() + 0.5, pt.y(), pt.z() + 0.5);
    }
}

=== FILE: Bootboatfloat/BootboatSpawn/src/test/java/ru/bootboat/spawn/SpawnPickerTest.java ===
package ru.bootboat.spawn;

import org.junit.jupiter.api.Test;

import java.util.Random;

import static org.junit.jupiter.api.Assertions.*;

/** Тест-кейс 16 ТЗ: первый вход в ±1500, повторный — обычного спавна не меняет. */
class SpawnPickerTest {

    private static SpawnPicker.WorldView flat(int ground, boolean lava) {
        return new SpawnPicker.WorldView() {
            @Override
            public int maxHeight() {
                return 320;
            }

            @Override
            public int highestY(int x, int z) {
                return ground;
            }

            @Override
            public boolean solid(int x, int y, int z) {
                return y <= ground;
            }

            @Override
            public boolean passable(int x, int y, int z) {
                return y > ground;
            }

            @Override
            public boolean danger(int x, int y, int z) {
                return lava && y == ground;
            }
        };
    }

    @Test
    void t16_firstSpawnInsideZone() {
        for (long seed = 0; seed < 200; seed++) {
            var p = SpawnPicker.pick(flat(64, false), new Random(seed), 0, 0, 1500, 40);
            assertNotNull(p, "seed=" + seed);
            assertTrue(Math.abs(p.x()) <= 1500, "x=" + p.x());
            assertTrue(Math.abs(p.z()) <= 1500, "z=" + p.z());
            assertEquals(65, p.y());
        }
    }

    @Test
    void t16_repeatJoinNotSpawned() {
        assertTrue(SpawnPicker.shouldSpawn(false));
        assertFalse(SpawnPicker.shouldSpawn(true));
    }

    @Test
    void t16_unsafeWorldReturnsNull() {
        assertNull(SpawnPicker.pick(flat(64, true), new Random(1), 0, 0, 1500, 40));
    }

    @Test
    void t16_noRoomUnderCeiling() {
        assertNull(SpawnPicker.pick(flat(318, false), new Random(1), 0, 0, 1500, 40));
    }

    @Test
    void t16_radiusZeroIsCenter() {
        var p = SpawnPicker.pick(flat(64, false), new Random(7), 100, -200, 0, 5);
        assertNotNull(p);
        assertEquals(100, p.x());
        assertEquals(-200, p.z());
    }
}

=== BUILD ===
cd Bootboatfloat/BootboatSpawn
mvn -B -ntp test        # ожидание: Tests run: 5, Failures: 0, Errors: 0
mvn -B -ntp package     # target/bootboat-spawn-1.0.0.jar

Если paper-api 26.3-R0.1-SNAPSHOT не резолвится: открыть
https://repo.papermc.io/repository/maven-public/io/papermc/paper/paper-api/
взять точную строку версии, подставить в pom, изменение записать в отчёт.
Если Paper отвергнет api-version '26.3' — он напечатает допустимые значения в лог,
подставить их, изменение записать в отчёт. Больше ничего не менять.

=== INSTALL ===
jar в plugins/ сервера, перезапуск. config.yml создастся сам.
Для одиночного теста радиус можно снизить до 200 — тогда спавн рядом с 0;0
и не надо бежать через всю зону. Перед открытием вернуть 1500.

=== PROMPT FOR CODER ===
Репозиторий: Bizonrelax/Sigmawillow, ветка main.
Прочитай Bootboatfloat/ARCHIVE-01-Spawn.md.
Создай файлы ровно по путям из маркеров FILE, содержимое бери целиком.
Собери: mvn -B -ntp test, затем mvn -B -ntp package.
Ожидание: 5 тестов зелёные, jar bootboat-spawn-1.0.0.jar.
Работай прямо в main, без веток и PR, минимум вызовов инструментов.
Пуш только после зелёного цикла. Jar приложи к отчёту файлом.
Отчёт: роль, шаги, число тестов, хеш коммита, вердикт ПУБЛИКОВАТЬ/НЕ ПУБЛИКОВАТЬ.
Если правил pom или api-version — напиши что и почему.
Ничего кроме этих файлов не создавать и не менять.

=== ACCEPTANCE ===
[ ] 5/5 тестов зелёные, skipped 0
[ ] jar собран и приложен к отчёту
[ ] в main один коммит, git status чистый
[ ] pom и plugin.yml не изменены, либо изменение объяснено в отчёте
[ ] посторонние файлы не тронуты
