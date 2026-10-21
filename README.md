name: Compilar Herobrine
on:
  push:
  workflow_dispatch:
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '8'
      - name: Criar arquivos do mod
        run: |
          mkdir -p $(dirname build.gradle)
          cat > build.gradle <<'EOF'
          buildscript {
              repositories {
                  jcenter()
                  maven { name = "forge"; url = "https://maven.minecraftforge.net" }
              }
              dependencies {
                  classpath 'net.minecraftforge.gradle:ForgeGradle:2.3-SNAPSHOT'
              }
          }
          apply plugin: 'net.minecraftforge.gradle.forge'

          version = "1.0"
          group = "com.herobrine"
          archivesBaseName = "herobrine-watcher"

          sourceCompatibility = targetCompatibility = '1.8'
          compileJava.sourceCompatibility = compileJava.targetCompatibility = '1.8'

          minecraft {
              version = "1.12.2-14.23.5.2854"
              runDir = "run"
              mappings = "stable_39"
          }

          processResources {
              inputs.property "version", project.version
              from(sourceSets.main.resources.srcDirs) {
                  include 'mcmod.info'
                  expand 'version': project.version
              }
              from(sourceSets.main.resources.srcDirs) {
                  exclude 'mcmod.info'
              }
          }
          EOF
          mkdir -p $(dirname src/main/resources/mcmod.info)
          cat > src/main/resources/mcmod.info <<'EOF'
          [{
            "modid": "herobrinewatcher",
            "name": "Herobrine Watcher",
            "description": "Herobrine rarely appears far away and just watches. He never attacks.",
            "version": "${version}",
            "mcversion": "1.12.2",
            "authorList": []
          }]
          EOF
          mkdir -p $(dirname src/main/resources/assets/herobrinewatcher/lang/en_us.lang)
          cat > src/main/resources/assets/herobrinewatcher/lang/en_us.lang <<'EOF'
          entity.herobrine.name=Herobrine
          EOF
          mkdir -p $(dirname src/main/java/com/herobrine/HerobrineMod.java)
          cat > src/main/java/com/herobrine/HerobrineMod.java <<'EOF'
          package com.herobrine;

          import com.herobrine.client.ClientSetup;
          import com.herobrine.entity.EntityHerobrine;
          import net.minecraft.util.ResourceLocation;
          import net.minecraftforge.common.MinecraftForge;
          import net.minecraftforge.common.config.Configuration;
          import net.minecraftforge.fml.common.Mod;
          import net.minecraftforge.fml.common.event.FMLPreInitializationEvent;
          import net.minecraftforge.fml.common.registry.EntityRegistry;
          import net.minecraftforge.fml.relauncher.Side;

          @Mod(modid = HerobrineMod.MODID, name = "Herobrine Watcher", version = "1.0", acceptedMinecraftVersions = "[1.12.2]")
          public class HerobrineMod {
              public static final String MODID = "herobrinewatcher";

              @Mod.Instance(MODID)
              public static HerobrineMod instance;

              // Configuravel em config/herobrinewatcher.cfg
              public static int spawnChance = 3000;     // chance 1 em N por segundo (por jogador)
              public static int watchDistance = 35;     // distancia em blocos
              public static int minDistance = 20;       // se o jogador chegar mais perto que isso, ele some
              public static int maxLifeSeconds = 45;    // tempo maximo visivel
              public static int stareSeconds = 3;       // segundos encarando ate ele sumir
              public static int cooldownMinutes = 20;   // intervalo minimo entre aparicoes

              @Mod.EventHandler
              public void preInit(FMLPreInitializationEvent event) {
                  Configuration cfg = new Configuration(event.getSuggestedConfigurationFile());
                  spawnChance = cfg.getInt("spawnChance", "general", spawnChance, 20, 1000000,
                          "Chance de 1 em N, checada 1x por segundo por jogador. Maior = mais raro.");
                  watchDistance = cfg.getInt("watchDistance", "general", watchDistance, 10, 60,
                          "Distancia (blocos) em que ele aparece observando.");
                  minDistance = cfg.getInt("minDistance", "general", minDistance, 2, 50,
                          "Se voce chegar mais perto que isso, ele desaparece.");
                  maxLifeSeconds = cfg.getInt("maxLifeSeconds", "general", maxLifeSeconds, 5, 600,
                          "Tempo maximo que ele fica la.");
                  stareSeconds = cfg.getInt("stareSeconds", "general", stareSeconds, 1, 60,
                          "Segundos olhando direto para ele ate ele sumir.");
                  cooldownMinutes = cfg.getInt("cooldownMinutes", "general", cooldownMinutes, 0, 600,
                          "Tempo minimo entre aparicoes para o mesmo jogador.");
                  if (cfg.hasChanged()) cfg.save();

                  EntityRegistry.registerModEntity(new ResourceLocation(MODID, "herobrine"),
                          EntityHerobrine.class, "herobrine", 0, instance, 64, 1, true);
                  MinecraftForge.EVENT_BUS.register(new SpawnHandler());

                  if (event.getSide() == Side.CLIENT) {
                      ClientSetup.init();
                  }
              }
          }
          EOF
          mkdir -p $(dirname src/main/java/com/herobrine/SpawnHandler.java)
          cat > src/main/java/com/herobrine/SpawnHandler.java <<'EOF'
          package com.herobrine;

          import com.herobrine.entity.EntityHerobrine;
          import net.minecraft.entity.player.EntityPlayer;
          import net.minecraft.util.math.BlockPos;
          import net.minecraft.util.math.MathHelper;
          import net.minecraft.util.math.Vec3d;
          import net.minecraft.world.World;
          import net.minecraftforge.fml.common.eventhandler.SubscribeEvent;
          import net.minecraftforge.fml.common.gameevent.TickEvent;
          import net.minecraftforge.fml.relauncher.Side;

          import java.util.HashMap;
          import java.util.Map;
          import java.util.UUID;

          public class SpawnHandler {
              private final Map<UUID, Long> lastSpawn = new HashMap<>();

              @SubscribeEvent
              public void onPlayerTick(TickEvent.PlayerTickEvent e) {
                  if (e.phase != TickEvent.Phase.END || e.side != Side.SERVER) return;
                  EntityPlayer p = e.player;
                  World w = p.world;
                  if (p.ticksExisted % 20 != 0 || p.isSpectator()) return;
                  if (w.provider.getDimension() != 0) return; // so no Overworld

                  Long last = lastSpawn.get(p.getUniqueID());
                  if (last != null && w.getTotalWorldTime() - last < HerobrineMod.cooldownMinutes * 1200L) return;

                  if (w.rand.nextInt(HerobrineMod.spawnChance) != 0) return;

                  // ja existe um Herobrine por perto?
                  if (!w.getEntitiesWithinAABB(EntityHerobrine.class, p.getEntityBoundingBox().grow(120)).isEmpty()) return;

                  Vec3d look = new Vec3d(p.getLookVec().x, 0, p.getLookVec().z).normalize();

                  for (int attempt = 0; attempt < 8; attempt++) {
                      double angle = w.rand.nextDouble() * Math.PI * 2;
                      double dx = Math.cos(angle), dz = Math.sin(angle);

                      // nao aparece "do nada" na frente do jogador
                      if (look.x * dx + look.z * dz > 0.3) continue;

                      int x = MathHelper.floor(p.posX + dx * HerobrineMod.watchDistance);
                      int z = MathHelper.floor(p.posZ + dz * HerobrineMod.watchDistance);
                      if (!w.isBlockLoaded(new BlockPos(x, 64, z))) continue;

                      BlockPos pos = w.getTopSolidOrLiquidBlock(new BlockPos(x, 0, z));
                      if (w.getBlockState(pos.down()).getMaterial().isLiquid()) continue;
                      if (Math.abs(pos.getY() - p.posY) > 12) continue;
                      if (!w.canSeeSky(pos)) continue;

                      EntityHerobrine h = new EntityHerobrine(w);
                      h.setLocationAndAngles(x + 0.5, pos.getY(), z + 0.5, 0, 0);
                      if (!w.getCollisionBoxes(h, h.getEntityBoundingBox()).isEmpty()) continue;

                      w.spawnEntity(h);
                      lastSpawn.put(p.getUniqueID(), w.getTotalWorldTime());
                      return;
                  }
              }
          }
          EOF
          mkdir -p $(dirname src/main/java/com/herobrine/entity/EntityHerobrine.java)
          cat > src/main/java/com/herobrine/entity/EntityHerobrine.java <<'EOF'
          package com.herobrine.entity;

          import com.herobrine.HerobrineMod;
          import net.minecraft.entity.EntityCreature;
          import net.minecraft.entity.SharedMonsterAttributes;
          import net.minecraft.entity.player.EntityPlayer;
          import net.minecraft.init.SoundEvents;
          import net.minecraft.nbt.NBTTagCompound;
          import net.minecraft.util.DamageSource;
          import net.minecraft.util.SoundEvent;
          import net.minecraft.util.math.Vec3d;
          import net.minecraft.world.World;

          import javax.annotation.Nullable;

          /** Herobrine: nunca ataca, nao se move, so observa e some. */
          public class EntityHerobrine extends EntityCreature {
              private int stareTicks = 0;

              public EntityHerobrine(World world) {
                  super(world);
                  setSize(0.6F, 1.8F);
                  setSilent(true);
              }

              @Override
              protected void applyEntityAttributes() {
                  super.applyEntityAttributes();
                  getEntityAttribute(SharedMonsterAttributes.MAX_HEALTH).setBaseValue(20.0D);
                  getEntityAttribute(SharedMonsterAttributes.MOVEMENT_SPEED).setBaseValue(0.0D);
                  getEntityAttribute(SharedMonsterAttributes.FOLLOW_RANGE).setBaseValue(64.0D);
              }

              // Sem tasks de IA: nao anda, nao ataca.

              @Override
              public void onUpdate() {
                  super.onUpdate();
                  if (world.isRemote) return;

                  EntityPlayer p = world.getClosestPlayerToEntity(this, 100.0D);
                  if (p == null || ticksExisted > HerobrineMod.maxLifeSeconds * 20) {
                      vanish();
                      return;
                  }

                  getLookHelper().setLookPositionWithEntity(p, 30.0F, 30.0F);

                  if (getDistance(p) < HerobrineMod.minDistance) {
                      vanish();
                      return;
                  }

                  // se o jogador encarar direto por alguns segundos, ele some
                  Vec3d look = p.getLook(1.0F).normalize();
                  Vec3d to = new Vec3d(posX - p.posX, (posY + 1.6D) - (p.posY + p.getEyeHeight()), posZ - p.posZ).normalize();
                  if (look.dotProduct(to) > 0.99D && p.canEntityBeSeen(this)) {
                      stareTicks++;
                  } else if (stareTicks > 0) {
                      stareTicks--;
                  }
                  if (stareTicks > HerobrineMod.stareSeconds * 20) {
                      vanish();
                  }
              }

              private void vanish() {
                  setDead();
              }

              @Override
              public boolean attackEntityFrom(DamageSource source, float amount) {
                  if (!world.isRemote) vanish(); // qualquer golpe faz ele sumir
                  return false;
              }

              @Override public boolean canBePushed() { return false; }
              @Override protected boolean canDespawn() { return false; }

              /** Nao e salvo no mundo: nunca reaparece ao recarregar. */
              @Override
              public boolean writeToNBTOptional(NBTTagCompound compound) {
                  return false;
              }

              @Nullable @Override protected SoundEvent getAmbientSound() { return null; }
              @Override protected SoundEvent getHurtSound(DamageSource s) { return SoundEvents.ENTITY_GENERIC_HURT; }
              @Nullable @Override protected SoundEvent getDeathSound() { return null; }
          }
          EOF
          mkdir -p $(dirname src/main/java/com/herobrine/client/RenderHerobrine.java)
          cat > src/main/java/com/herobrine/client/RenderHerobrine.java <<'EOF'
          package com.herobrine.client;

          import com.herobrine.HerobrineMod;
          import com.herobrine.entity.EntityHerobrine;
          import net.minecraft.client.model.ModelPlayer;
          import net.minecraft.client.renderer.entity.RenderLiving;
          import net.minecraft.client.renderer.entity.RenderManager;
          import net.minecraft.util.ResourceLocation;

          public class RenderHerobrine extends RenderLiving<EntityHerobrine> {
              private static final ResourceLocation TEXTURE =
                      new ResourceLocation(HerobrineMod.MODID, "textures/entity/herobrine.png");

              public RenderHerobrine(RenderManager manager) {
                  super(manager, new ModelPlayer(0.0F, false), 0.5F);
              }

              @Override
              protected ResourceLocation getEntityTexture(EntityHerobrine entity) {
                  return TEXTURE;
              }
          }
          EOF
          mkdir -p $(dirname src/main/java/com/herobrine/client/ClientSetup.java)
          cat > src/main/java/com/herobrine/client/ClientSetup.java <<'EOF'
          package com.herobrine.client;

          import com.herobrine.entity.EntityHerobrine;
          import net.minecraftforge.fml.client.registry.RenderingRegistry;
          import net.minecraftforge.fml.relauncher.Side;
          import net.minecraftforge.fml.relauncher.SideOnly;

          @SideOnly(Side.CLIENT)
          public class ClientSetup {
              public static void init() {
                  RenderingRegistry.registerEntityRenderingHandler(EntityHerobrine.class, RenderHerobrine::new);
              }
          }
          EOF
          mkdir -p src/main/resources/assets/herobrinewatcher/textures/entity
          cat > textura.py <<'EOF'
          from PIL import Image
          im = Image.new("RGBA",(64,64),(0,0,0,0))
          px = im.load()
          def fill(x,y,w,h,c):
              for i in range(x,x+w):
                  for j in range(y,y+h):
                      px[i,j]=c
          def box(x,y,w,h,d,top,bot,side,front,back=None):
              back = back or side
              fill(x+d,y,w,d,top); fill(x+d+w,y,w,d,bot)
              fill(x,y+d,d,h,side); fill(x+d,y+d,w,h,front)
              fill(x+d+w,y+d,d,h,side); fill(x+2*d+w,y+d,w,h,back)
          skin=(198,142,104,255); skin2=(178,124,90,255)
          hair=(60,38,20,255); shirt=(0,170,170,255); shirt2=(0,140,140,255)
          pants=(50,50,160,255); pants2=(40,40,130,255); shoe=(90,90,90,255)
          W=(255,255,255,255)
          box(0,0,8,8,8,hair,skin2,hair,skin,hair)
          fill(8,8,8,2,hair)
          fill(8,10,1,2,hair); fill(15,10,1,2,hair)
          fill(9,13,2,1,W); fill(13,13,2,1,W)
          fill(11,15,2,1,(120,70,50,255))
          box(16,16,8,12,4,shirt,shirt2,shirt2,shirt)
          for (ax,ay) in ((40,16),(32,48)):
              box(ax,ay,4,12,4,shirt,skin2,shirt2,shirt)
              fill(ax+4,ay+4+9,4,3,skin); fill(ax,ay+4+9,4,3,skin2)
              fill(ax+8,ay+4+9,4,3,skin2); fill(ax+12,ay+4+9,4,3,skin2)
          for (lx,ly) in ((0,16),(16,48)):
              box(lx,ly,4,12,4,pants,shoe,pants2,pants)
              fill(lx+4,ly+4+10,4,2,shoe); fill(lx,ly+4+10,12+4,2,shoe)
          im.save("src/main/resources/assets/herobrinewatcher/textures/entity/herobrine.png")
          EOF
          pip install pillow
          python3 textura.py
      - name: Baixar MDK do Forge e juntar
        run: |
          curl -fL -o mdk.zip https://maven.minecraftforge.net/net/minecraftforge/forge/1.12.2-14.23.5.2854/forge-1.12.2-14.23.5.2854-mdk.zip
          mkdir mdk
          unzip -q mdk.zip -d mdk
          rm -rf mdk/src
          cp -r src mdk/src
          cp build.gradle mdk/build.gradle
      - name: Compilar
        working-directory: mdk
        run: |
          chmod +x gradlew
          ./gradlew setupCiWorkspace build --no-daemon
      - uses: actions/upload-artifact@v4
        with:
          name: herobrine-watcher
          path: mdk/build/libs/*.jar
