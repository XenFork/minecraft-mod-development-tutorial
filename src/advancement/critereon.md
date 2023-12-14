# 进度触发器

1. 触发方式
2. 实现原理

# 触发方式
* 它使用的是trigger进行触发
* 我们先看看mojang的代码 

<details>
<summary>展开查看代码 net.minecraft.advancements.critereon.RecipeUnlockedTrigger</summary>
<pre>
<code>
package net.minecraft.advancements.critereon;

import com.google.gson.JsonObject;
import net.minecraft.advancements.critereon.EntityPredicate.Composite;
import net.minecraft.resources.ResourceLocation;
import net.minecraft.server.level.ServerPlayer;
import net.minecraft.util.GsonHelper;
import net.minecraft.world.item.crafting.Recipe;

public class RecipeUnlockedTrigger extends SimpleCriterionTrigger<TriggerInstance> {
static final ResourceLocation ID = new ResourceLocation("recipe_unlocked");

    public RecipeUnlockedTrigger() {
    }

    public ResourceLocation getId() {
        return ID;
    }

    public TriggerInstance createInstance(JsonObject json, EntityPredicate.Composite entityPredicate, DeserializationContext conditionsParser) {
        ResourceLocation resourceLocation = new ResourceLocation(GsonHelper.getAsString(json, "recipe"));
        return new TriggerInstance(entityPredicate, resourceLocation);
    }

    public void trigger(ServerPlayer player, Recipe<?> recipe) {
        this.trigger(player, (arg2) -> arg2.matches(recipe));
    }

    public static TriggerInstance unlocked(ResourceLocation recipe) {
        return new TriggerInstance(Composite.ANY, recipe);
    }

    public static class TriggerInstance extends AbstractCriterionTriggerInstance {
        private final ResourceLocation recipe;

        public TriggerInstance(EntityPredicate.Composite player, ResourceLocation recipe) {
            super(RecipeUnlockedTrigger.ID, player);
            this.recipe = recipe;
        }

        public JsonObject serializeToJson(SerializationContext context) {
            JsonObject jsonObject = super.serializeToJson(context);
            jsonObject.addProperty("recipe", this.recipe.toString());
            return jsonObject;
        }

        public boolean matches(Recipe<?> recipe) {
            return this.recipe.equals(recipe.getId());
        }
    }
}
</code>
</pre>
</details>

* 它被分为  SimpleCriterionTrigger<T extends AbstractCriterionTriggerInstance> 和 AbstractCriterionTriggerInstance 两个类
* 其中我们需要做的就是
* 1. 写自己的trigger 和 matches
* 2. 在服务器启动前将触发器注册
* 3. 实现触发

## 接下来我们来实现配方合成时触发成就
1. 写一个总触发器 CraftingRecipeTrigger类并且创建CraftingRecipeTrigger内部子类 TriggerInstance类
2. TriggerInstance需要实现序列化和创建对象
3. 而CraftingRecipeTrigger所需要做的就是解析并反序列化创建实例 createInstance
4. 在TriggerInstance实现matches判断
5. 在CraftingRecipeTrigger实现trigger触发器
6. trigger触发器使用SimpleCriterionTrigger里面的trigger(ServerPlayer, Predicate<? extends AbstractCriterionTriggerInstance>)
7. 注册实例(forge,fabric,architectury)

```java
import com.google.gson.JsonObject;
import net.minecraft.advancements.critereon.*;
import net.minecraft.resources.ResourceLocation;
import net.minecraft.world.item.ItemStack;
import org.jetbrains.annotations.NotNull;

public class CraftingRecipeTrigger extends SimpleCriterionTrigger<CraftingRecipeTrigger.TriggerInstance> {

    static final ResourceLocation ID = new ResourceLocation("tutorial", "crafting_recipe");

    @Override

    protected @NotNull TriggerInstance createInstance(@NotNull JsonObject json, EntityPredicate.@NotNull Composite player, @NotNull DeserializationContext context) {
        return new TriggerInstance(player, ItemPredicate.fromJson(json.get("item")));
    }

    public void trigger(ServerPlayer player, ItemStack item) {
        this.trigger(player, arg -> arg.matches(item));

    }

    // ... 1
    public static class TriggerInstance extends AbstractCriterionTriggerInstance {

        private final ItemPredicate item;

        public TriggerInstance(EntityPredicate.Composite player, ItemPredicate item) {
            super(CraftingRecipeTrigger.ID, player);
            this.item = item;
        }

        @Override
        public @NotNull JsonObject serializeToJson(@NotNull SerializationContext context) {
            JsonObject jsonObject = super.serializeToJson(context);
            jsonObject.add("item", this.item.serializeToJson());
            return jsonObject;
        }

        public boolean matches(ItemStack item) {
            return this.item.matches(item);
        }
    }
}
```


## forge
在主类中实现

```java
import net.minecraftforge.event.entity.player.PlayerEvent;
import net.minecraftforge.fml.common.Mod;

@Mod("tutorial")
public class Tutorial {
    public static final CraftingRecipeTrigger cr = new CraftingRecipeTrigger();

    public Tutorial() {

    }

    @SubscribeEvent
    public static void craftEvents(PlayerEvent.ItemCraftedEvent event) {
        if (event.getEntity() instanceof ServerPlayer serverPlayer) {
            cr.trigger(serverPlayer, event.getCrafting());// 触发
        }
    }
}
```

## Fabric
在主类中实现

```java

import net.fabricmc.api.ModInitializer;

public class Tutorial implements ModInitializer {
    public static final CraftingRecipeTrigger cr = new CraftingRecipeTrigger();

    @Override
    public void onInitialize() {
        
    }
}
```

```java
import net.minecraft.world.inventory.ResultSlot;
import org.spongepowered.asm.mixin.Mixin;
import org.spongepowered.asm.mixin.injection.At;
import org.spongepowered.asm.mixin.injection.Inject;

@Mixin(ResultSlot.class)
public class MixinCraftingRecipe {
    @Inject(
            method = "checkTakeAchievements",
            at = @At(value = "INVOKE",
                    target = "Lnet/minecraft/world/item/ItemStack;onCraftedBy(Lnet/minecraft/world/level/Level;Lnet/minecraft/world/entity/player/Player;I)V",
                    shift = At.Shift.AFTER)
    )
    private void craft(ItemStack itemStack, CallbackInfo ci) {
        Tutorial.cr.trigger(serverPlayer, itemStack);
    }
    
}
```

## Architectury
多加载器模式触发的事件

```java
import dev.architectury.event.events.common.PlayerEvent;

public class Tutorial {
    public static final CraftingRecipeTrigger cr = new CraftingRecipeTrigger();

    public static void init() {
        PlayerEvent.CRAFT_ITEM.register((player, constructed, inventory) -> {
            cr.trigger(serverPlayer, constructed);
        });
    }
}
```