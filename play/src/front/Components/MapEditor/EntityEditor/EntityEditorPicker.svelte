<script lang="ts">
    import type { EntityPrefab } from "@workadventure/map-editor";
    import { onDestroy } from "svelte";
    import { get } from "svelte/store";
    import { LL } from "../../../../i18n/i18n-svelte";
    import { gameManager } from "../../../Phaser/Game/GameManager";
    import { EntityVariant } from "../../../Phaser/Game/MapEditor/Entities/EntityVariant";
    import {
        mapEditorDeleteCustomEntityEventStore,
        mapEditorEntityModeStore,
        mapEditorModifyCustomEntityEventStore,
        mapEditorSelectedEntityPrefabStore,
        mapEditorSelectedEntityStore,
        SelectableTag,
        selectCategoryStore,
    } from "../../../Stores/MapEditorStore";
    import CustomEntityEditionForm from "./CustomEntityEditionForm/CustomEntityEditionForm.svelte";
    import EntitiesGrid from "./EntitiesGrid.svelte";
    import EntityImage from "./EntityItem/EntityImage.svelte";
    import EntityVariantColorPicker from "./EntityItem/EntityVariantColorPicker.svelte";
    import EntityVariantPositionPicker from "./EntityItem/EntityVariantPositionPicker.svelte";
    import EntityUpload from "./EntityUpload/EntityUpload.svelte";
    import TagListItem from "./TagListItem.svelte";
    import { IconChevronLeft, IconDeselect, IconPencil } from "@wa-icons";

    const entitiesCollectionsManager = gameManager.getCurrentGameScene().getEntitiesCollectionsManager();
    const entitiesPrefabsVariants = entitiesCollectionsManager.getEntitiesPrefabsVariantStore();

    let pickedEntity: EntityPrefab | undefined = undefined;
    let pickedEntityVariant: EntityVariant | undefined = undefined;
    let selectedColor: string;

    let searchTerm = "";

    const mapEditorSelectedEntityPrefabStoreUnsubscriber = mapEditorSelectedEntityPrefabStore.subscribe(
        (prefab?: EntityPrefab) => {
            pickedEntity = prefab;
        }
    );

    const entitiesPrefabsVariantStoreUnsubscriber = entitiesCollectionsManager
        .getEntitiesPrefabsVariantStore()
        .subscribe((entitiesPrefabsVariants) => {
            if (pickedEntityVariant) {
                pickedEntityVariant = entitiesPrefabsVariants.find(
                    (entityPrefabVariant) => pickedEntityVariant?.id === entityPrefabVariant.id
                );
                pickedEntity = pickedEntityVariant?.defaultPrefab;
            }
        });

    function removeEntity(id: string) {
        mapEditorDeleteCustomEntityEventStore.set({ id });
        clearEntitySelection();
        setIsEditingCustomEntity(false);
    }

    function saveCustomEntityModifications(customEntity: EntityPrefab) {
        mapEditorModifyCustomEntityEventStore.set({
            ...customEntity,
        });
        setIsEditingCustomEntity(false);
    }

    function onPickItem(entityPrefab: EntityPrefab) {
        mapEditorSelectedEntityPrefabStore.set(entityPrefab);
    }

    function onPickEntityVariant(entityVariant: EntityVariant) {
        pickedEntity = entityVariant.defaultPrefab;
        pickedEntityVariant = entityVariant;
        onColorChange(pickedEntity.color);
    }

    function onColorChange(color: string) {
        selectedColor = color;
        pickedEntity = pickedEntityVariant?.getEntityPrefabsPositions(color)[0];
        mapEditorSelectedEntityPrefabStore.set(pickedEntity);
    }

    function onSelectedTag(tag: string) {
        selectCategoryStore.set(tag);
    }

    function displayTagListAndClearCurrentSelection() {
        get(mapEditorSelectedEntityStore)?.delete();
        mapEditorEntityModeStore.set("ADD");
        clearEntitySelection();
        selectCategoryStore.set(undefined);
        searchTerm = "";
        setIsEditingCustomEntity(false);
    }

    function clearEntitySelection() {
        pickedEntityVariant = undefined;
        pickedEntity = undefined;
        mapEditorSelectedEntityStore.set(undefined);
        mapEditorSelectedEntityPrefabStore.set(undefined);
    }

    let isEditingCustomEntity = false;
    function setIsEditingCustomEntity(isEditing: boolean) {
        isEditingCustomEntity = isEditing;
    }

    function getEntitiesPrefabsVariantsGroupedByTagWithCustomFirst(entitiesPrefabsVariants: EntityVariant[]): {
        [tag: string]: EntityVariant[];
    } {
        const entitiesPrefabsVariantsGroupedByTag = entitiesPrefabsVariants.reduce(
            (groupByTag: { [tag: string]: EntityVariant[] }, entityPrefabVariant) => {
                const { tags } = entityPrefabVariant.defaultPrefab;
                tags.forEach((tag) => {
                    groupByTag[tag] = groupByTag[tag] ?? [];
                    groupByTag[tag].push(entityPrefabVariant);
                });
                return groupByTag;
            },
            {}
        );
        const customEntitiesPrefabsVariants = {
            Custom: entitiesPrefabsVariants.filter(
                (entityPrefabVariant) => entityPrefabVariant.defaultPrefab.type === "Custom"
            ),
        };
        return {
            ...customEntitiesPrefabsVariants,
            ...Object.fromEntries(Object.entries(entitiesPrefabsVariantsGroupedByTag).sort()),
        };
    }

    function getEntitiesPrefabsVariantsFilteredByTag(
        entitiesPrefabsVariants: EntityVariant[],
        tag: SelectableTag,
        searchTerm: string
    ) {
        if (tag === undefined) {
            return entitiesPrefabsVariants.filter(
                (entityPrefabVariant) =>
                    entityPrefabVariant.defaultPrefab.tags
                        .join(",")
                        .toLocaleLowerCase()
                        .indexOf(searchTerm.toLocaleLowerCase()) != -1 ||
                    entityPrefabVariant.defaultPrefab.name.toLowerCase().includes(searchTerm.toLowerCase())
            );
        }
        if ($selectCategoryStore === "Custom") {
            return entitiesPrefabsVariants.filter(
                (entityPrefabVariant) =>
                    entityPrefabVariant.defaultPrefab.type === "Custom" &&
                    entityPrefabVariant.defaultPrefab.name.toLowerCase().includes(searchTerm)
            );
        }
        return entitiesPrefabsVariants.filter(
            (entityPrefabVariant) =>
                entityPrefabVariant.defaultPrefab.tags.includes(tag) &&
                entityPrefabVariant.defaultPrefab.name.toLowerCase().includes(searchTerm)
        );
    }

    onDestroy(() => {
        mapEditorSelectedEntityPrefabStoreUnsubscriber();
        entitiesPrefabsVariantStoreUnsubscriber();
    });
</script>

<div class="tw-flex tw-flex-col tw-flex-1 tw-overflow-auto tw-gap-2">
    <div class="tw-flex tw-flex-col tw-gap-2">
        <div>
            {#if $selectCategoryStore === undefined}
                <p class="tw-text-[22px] tw-m-0">{$LL.mapEditor.entityEditor.header.title()}</p>
                <p class="tw-m-0 tw-opacity-50">{$LL.mapEditor.entityEditor.header.description()}</p>
            {:else}
                <button
                    class="tw-p-0"
                    data-testid="clearCurrentSelection"
                    on:click={displayTagListAndClearCurrentSelection}
                    ><IconChevronLeft />{$LL.mapEditor.entityEditor.buttons.back()} - {$selectCategoryStore
                        ? $selectCategoryStore
                        : ""}</button
                >
            {/if}
        </div>
        <div class="tw-flex">
            <input
                class="tw-flex-1 tw-h-8 !tw-border-solid !tw-rounded-2xl !tw-border-gray-400 !tw-placeholder-gray-400"
                type="search"
                bind:value={searchTerm}
                placeholder={$LL.mapEditor.entityEditor.itemPicker.searchPlaceholder()}
            />
        </div>
    </div>
    <div class={`tw-flex-1 tw-overflow-auto ${pickedEntity ? "tw-pt-44" : ""}`}>
        {#if $selectCategoryStore === undefined && searchTerm === ""}
            <ul class="tw-list-none !tw-p-0 tw-min-w-full">
                {#each Object.entries(getEntitiesPrefabsVariantsGroupedByTagWithCustomFirst($entitiesPrefabsVariants)) as [tag, entitiesPrefabsVariants] (tag)}
                    <TagListItem
                        on:onSelectedTag={(event) => {
                            onSelectedTag(event.detail);
                        }}
                        {tag}
                        {entitiesPrefabsVariants}
                    />
                {/each}
            </ul>
        {:else}
            {#if pickedEntityVariant && pickedEntity}
                <div
                    class="tw-flex tw-flex-row tw-gap-2 tw-items-center tw-justify-center tw-border-b-blue-50 tw-mb-2 tw-min-h-[200px] tw-absolute tw-bg-dark-purple/90 tw-rounded-2xl tw-w-full tw-left-0 tw-top-24"
                >
                    {#if isEditingCustomEntity}
                        <CustomEntityEditionForm
                            customEntity={pickedEntity}
                            on:closeForm={() => {
                                setIsEditingCustomEntity(false);
                            }}
                            on:removeEntity={({ detail: { entityId } }) => {
                                removeEntity(entityId);
                            }}
                            on:applyEntityModifications={({ detail: customModifiedEntity }) =>
                                saveCustomEntityModifications(customModifiedEntity)}
                        />
                    {:else}
                        <EntityImage
                            classNames="tw-h-16 tw-w-[64px] tw-object-contain"
                            imageSource={pickedEntity.imagePath}
                            imageAlt={pickedEntity.name}
                        />
                        <div>
                            <p class="tw-m-0"><b>{pickedEntityVariant.defaultPrefab.name}</b></p>
                            <EntityVariantColorPicker
                                colors={pickedEntityVariant.colors}
                                {selectedColor}
                                {onColorChange}
                            />
                            <EntityVariantPositionPicker
                                entityPrefabsPositions={pickedEntityVariant.getEntityPrefabsPositions(selectedColor)}
                                selectedEntity={pickedEntity}
                                {onPickItem}
                            />
                        </div>
                        {#if pickedEntity.type === "Custom"}
                            <button
                                class="tw-bg-blue-500 tw-rounded"
                                data-testid="editEntity"
                                on:click={() => setIsEditingCustomEntity(true)}
                                ><IconPencil font-size={16} />{$LL.mapEditor.entityEditor.buttons.editEntity()}</button
                            >
                        {/if}
                        <button
                            class="tw-self-start tw-absolute tw-right-0"
                            data-testid="clearEntitySelection"
                            on:click={clearEntitySelection}><IconDeselect font-size={20} /></button
                        >
                    {/if}
                </div>
            {/if}
            {#if !isEditingCustomEntity}
                <EntitiesGrid
                    entityPrefabVariants={getEntitiesPrefabsVariantsFilteredByTag(
                        $entitiesPrefabsVariants,
                        $selectCategoryStore,
                        searchTerm
                    )}
                    onSelectEntity={onPickEntityVariant}
                    currentSelectedEntityId={pickedEntity?.id}
                />
            {/if}
        {/if}
    </div>
    {#if pickedEntity === undefined}
        <EntityUpload />
    {/if}
</div>
