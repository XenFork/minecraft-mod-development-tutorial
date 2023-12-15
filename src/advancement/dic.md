# 进度存在代码中

1. 在代码生成一个进度对象
2. 在服务器启动正式启动前写进服务端的进度中去

## Forge

```java
import net.minecraft.advancements.Advancement;
import net.minecraft.advancements.critereon.InventoryChangeTrigger;
import net.minecraft.resources.ResourceLocation;
import net.minecraft.server.MinecraftServer;
import net.minecraft.world.item.Items;
import net.minecraftforge.common.MinecraftForge;
import net.minecraftforge.event.server.ServerAboutToStartEvent;
import net.minecraftforge.eventbus.api.SubscribeEvent;
import net.minecraftforge.fml.common.Mod;

@Mod("tutorial")
public class Tutorial {

    public static final Advancement root =
            Advancement.Builder.advancement()
                    .addCriterion("key", InventoryChangeTrigger.TriggerInstance.hasItems(Items.STICK))
                    .build(new ResourceLocation("tutorial", "main/root"));

    public Tutorial() {
        MinecraftForge.EVENT_BUS.register(this);
    }


    @SubscribeEvent
    public void aboutServerStart(ServerAboutToStartEvent event) {
        MinecraftServer server = event.getServer();
        server.getAdvancements().getAllAdvancements().add(root);
    }
}
```

## Fabric

```java
import net.fabricmc.api.ModInitializer;
import net.fabricmc.fabric.api.event.lifecycle.v1.ServerLifecycleEvents;

public class Tutorial implements ModInitializer {
    public static final Advancement root =
            Advancement.Builder.advancement()
                    .addCriterion("key", InventoryChangeTrigger.TriggerInstance.hasItems(Items.STICK))
                    .build(new ResourceLocation("tutorial", "main/root"));
    @Override
    public void onInitialize() {
        ServerLifecycleEvents.SERVER_STARTING.register(server -> {
            server.getAdvancements().getAllAdvancements().add(root);
        });
    }
}
```

## Architectury

```java
import dev.architectury.event.events.common.LifecycleEvent;

public class Tutorial {

    public static final Advancement root =
            Advancement.Builder.advancement()
                    .addCriterion("key", InventoryChangeTrigger.TriggerInstance.hasItems(Items.STICK))
                    .build(new ResourceLocation("tutorial", "main/root"));
    
    public static void init() {
        LifecycleEvent.SERVER_BEFORE_START.register(server -> {
            server.getAdvancements().getAllAdvancements().add(root);
        });
    }
}
```