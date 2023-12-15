# 进度数据生成器

1. 实现方式

## Mojang
这个是mojang的原生Advancement DataGen代码
```java

package net.minecraft.data.advancements;

import com.google.common.collect.ImmutableList;
import com.google.common.collect.Sets;
import com.mojang.logging.LogUtils;

import java.io.IOException;
import java.nio.file.Path;
import java.util.*;
import java.util.function.Consumer;

import net.minecraft.advancements.Advancement;
import net.minecraft.data.*;
import net.minecraft.data.DataGenerator.Target;
import net.minecraft.resources.ResourceLocation;
import net.minecraftforge.common.data.ExistingFileHelper;
import org.slf4j.Logger;

public class AdvancementProvider implements DataProvider {
    private static final Logger LOGGER = LogUtils.getLogger();
    private final DataGenerator.PathProvider pathProvider;
    private final List<Consumer<Consumer<Advancement>>> tabs = ImmutableList.of(new TheEndAdvancements(), new HusbandryAdvancements(), new AdventureAdvancements(), new NetherAdvancements(), new StoryAdvancements());
    protected ExistingFileHelper fileHelper;

    /** @deprecated */
    @Deprecated
    public AdvancementProvider(DataGenerator generator) {
        this.pathProvider = generator.createPathProvider(Target.DATA_PACK, "advancements");
    }

    public AdvancementProvider(DataGenerator generatorIn, ExistingFileHelper fileHelperIn) {
        this.pathProvider = generatorIn.createPathProvider(Target.DATA_PACK, "advancements");
        this.fileHelper = fileHelperIn;
    }

    public void run(CachedOutput output) {
        Set<ResourceLocation> set = Sets.newHashSet();
        Consumer<Advancement> consumer = (arg2) -> {
            if (!set.add(arg2.getId())) {
                throw new IllegalStateException("Duplicate advancement " + arg2.getId());
            } else {
                Path path = this.pathProvider.json(arg2.getId());

                try {
                    DataProvider.saveStable(output, arg2.deconstruct().serializeToJson(), path);
                } catch (IOException var6) {
                    LOGGER.error("Couldn't save advancement {}", path, var6);
                }

            }
        };
        this.registerAdvancements(consumer, this.fileHelper);
    }

    protected void registerAdvancements(Consumer<Advancement> consumer, ExistingFileHelper fileHelper) {

        for (Consumer<Consumer<Advancement>> tab : this.tabs) {
            consumer1.accept(consumer);
        }

    }

    public String getName() {
        return "Advancements";
    }
}


```


## Forge
1. 继承原版进度提供者
2. 使用access Transformer 改变field tags的私有类型或者使用mixin接口setTags。 我们使用的前者

```java
import net.minecraft.advancements.Advancement;
import net.minecraft.advancements.critereon.EntityPredicate;
import net.minecraft.advancements.critereon.ItemPredicate;
import net.minecraft.data.CachedOutput;
import net.minecraft.data.DataGenerator;
import net.minecraft.data.advancements.AdvancementProvider;
import net.minecraft.world.item.Items;

import java.util.List;
import java.util.function.Consumer;

/** @noinspection ALL*/
public class TutorialProvider extends AdvancementProvider {
    public TutorialProvider(DataGenerator generator) {
        super(generator);
    }

    public List<Consumer<Consumer<Advancement>>> getTags() {
        return List.of(TutorialProvider::register);
    }

    private static void register(Consumer<Advancement> advancementConsumer) {
        Advancement.Builder.advancement()
                .addCriterion(new CraftingRecipeTrigger.TriggerInstance(EntityPredicate.Composite.ANY, ItemPredicate.Builder.item().of(Items.STICK)));
    }

    @Override
    public void run(CachedOutput output) {
        tags = getTags();
        super.run(output);
    }

    @Override
    public String getName() {
        return "tutorial advancement provider";
    }
}
```

## Fabric

```java

```

## Architectury