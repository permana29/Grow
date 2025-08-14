local debug_ = false -- debug logs

while not game:IsLoaded() do
    task.wait()
end

local plrs:Players = game:GetService("Players")
while not plrs.LocalPlayer do
    task.wait()
end

local plr:Player = plrs.LocalPlayer

local LOAD_WAIT = 1

local function loadWait()
    while not plr:GetAttribute("Finished_Loading") do
        task.wait()
    end
    while not plr:GetAttribute("DataFullyLoaded") do
        task.wait()
    end
    while not plr:GetAttribute("Setup_Finished") do
        task.wait()
    end
    game:GetService("ReplicatedStorage").GameEvents.Finish_Loading:FireServer()
    game:GetService("ReplicatedStorage").GameEvents.LoadScreenEvent:FireServer(plr)
    while not plr:GetAttribute("Loading_Screen_Finished") do
        task.wait()
    end
    task.wait(LOAD_WAIT)
end

local dprint = debug_ and print or function() end

dprint("waiting for load")

if plr:GetAttribute("Loading_Screen_Finished") then
    game:GetService("ReplicatedStorage").GameEvents.Finish_Loading:FireServer()
    game:GetService("ReplicatedStorage").GameEvents.LoadScreenEvent:FireServer(plr)
else
    loadWait()
end

dprint("loaded")

local GAG = {}

local httpService = game:GetService("HttpService")
local rs = game:GetService("ReplicatedStorage")
local rf = game:GetService("ReplicatedFirst")
local gameEvents = rs:WaitForChild("GameEvents")
local modules = rs.Modules
local dataService = require(modules.DataService);
local notification = require(modules.Notification)
-- notification example 
--notification:CreateNotification("<font color=\"#ADD8E6\"><b>Your Seed Shop stock has been reset!</b></font>");
local seedData = require(rs.Data.SeedData)
local marketController = require(modules.MarketController)
local giftController = require(modules.GiftController)
local updateService = require(modules.UpdateService)
local inventoryService = require(modules.InventoryService)

local MAX_PLANTS_ON_FARM = 800 -- ?

local function setConsoleEnabled(bool:boolean)
    game:GetService("StarterGui"):SetCore("DevConsoleVisible", bool)
end

local function getHRP(): Instance
    while not plr.Character do
        task.wait()
    end
    return plr.Character:FindFirstChild("HumanoidRootPart") or plr.Character:WaitForChild("HumanoidRootPart",9e9)
end

local function round(num:number,decimals:number):number
    decimals = decimals or 0
    local power = 10^decimals
    return math.round(num*power)/power
end

type Mutation = {
    Name: string, -- artificial (key added from dict)
    ValueMulti: number,
    Color: Color3,
    _AddFX: ()->nil,
    _RemoveFX: ()->nil,
}

-- PLANTDATA CLASS --
type PlantData = {
    Name:string,
    _Age:NumberValue,
    Weight:NumberValue,
    _Weight:number,
    Age:number,
    MaxAge:number,
    FruitSizeMultiplier:number,
    FruitVariantLuck:number,
    GrowRateMulti:number,
    LuckFruitSizeMultiplier:number,
    LuckFruitVariantLuck:number,
    HasGrown: (self:PlantData) -> boolean,
    GetFruitsData: (self:PlantData) -> {FruitData},
    IsHarvestableWithoutFruit: (self:PlantData) -> boolean,
    _instance:Instance,
    -- those are only for plants harvestable without fruit, bamboo etc
    Weight:number,
    _Weight:NumberValue,
    Item_Seed:number,
    _Item_Seed:NumberValue,
    Variant:string,
    _Variant:StringValue
}
local PlantData:PlantData = {}
PlantData.__index = function(t,key)
    if table.find({ -- hack for NumberValue or StringValue plant properties so we can get their values dynamically without doing .Value with each access
        "Age",
        -- those are only for plants harvestable without fruit, bamboo etc
        "Weight",
        "Item_Seed",
        "Variant"
    },key) then
        return t["_"..key].Value
    elseif table.find({ -- hack for dynamically fetching attribute values
        "MaxAge",
        "FruitSizeMultiplier",
        "FruitVariantLuck",
        "GrowRateMulti",
        "LuckFruitSizeMultiplier",
        "LuckFruitVariantLuck"
    },key) then
        return t._instance:GetAttribute(key)
    elseif table.find({ -- hack for regular plant instance properties that may change for whatever reason
        "Name"
    },key) then
        return t._instance[key]
    else
        return rawget(PlantData,key) or rawget(t,key)
    end
end
-- PLANTDATA CLASS --

-- FRUITDATA CLASS --
type FruitData = {
    Name:string,
    FruitSpawnIndex:number,
    GrowRateMulti:number,
    _Age:NumberValue,
    Age:number,
    MaxAge:number,
    UUID:string, -- artificial (key added from dict)
    WeightMulti:number,
    _ItemSeed:NumberValue,
    ItemSeed:number,
    _Variant:StringValue,
    Variant:string,
    _Weight:NumberValue,
    Weight:number,
    HasGrown: (self:FruitData) -> boolean,
    GetPickupPrompt: (self:FruitData) -> ProximityPrompt,
    IsWithinPickupRange: (self:FruitData) -> boolean,
    GetMutations: (self:FruitData) -> {string},
    _instance:Instance
}
local FruitData:FruitData = {}
FruitData.__index = function(t,key)
    if table.find({ -- hack for NumberValue or StringValue properties so we can get their values dynamically without doing .Value with each access
        "Age",
        "ItemSeed",
        "Variant",
        "Weight",
    },key) then
        return t["_"..key].Value
    elseif table.find({ -- hack for dynamically fetching attribute values
        "GrowRateMulti",
        "MaxAge",
        "UUID",
        "WeightMulti",
    },key) then
        return t._instance:GetAttribute(key)
    elseif table.find({ -- hack for regular instance properties that may change for whatever reason
        "Name"
    },key) then
        return t._instance[key]
    else
        return rawget(FruitData,key) or rawget(t,key)
    end
end
-- FRUITDATA CLASS --

function PlantData:HasGrown(): boolean
    return self.Age >= self.MaxAge
end

function PlantData:IsHarvestableWithoutFruit(): boolean
    return self._instance:FindFirstChild("Fruits")==nil
end

function PlantData:GetFruitsData(): {FruitData}
    local fruitsData:{FruitData} = {}
    if self._instance:FindFirstChild("Fruits") then -- REGULAR FRUITS GROWING
        for _, fruitInstance in self._instance.Fruits:GetChildren() do
            table.insert(fruitsData, FruitData.new(fruitInstance))
        end
    elseif self:IsHarvestableWithoutFruit() then -- PLANTS HARVESTABLE BY THEMSELVES
        return {FruitData.new(table.copy(self))}
    end
    return fruitsData
end

function PlantData.new(plantInstance:Instance): PlantData
    local self:PlantData = setmetatable({}, PlantData)

    self._Age = plantInstance.Grow.Age
    self._instance = plantInstance

    -- IS THIS A GAME BUG OR SOMETHING BECAUSE THE PLANT ITSELF (NOT THE FRUIT) HAS WEIGHT?
    self._Weight = plantInstance.Weight
    self._Item_Seed = plantInstance.Item_Seed
    self._Variant = plantInstance.Variant

    return self
end

function FruitData:HasGrown(): boolean
    return self.Age >= self.MaxAge
end

function FruitData:GetPickupPrompt(): ProximityPrompt
    return self._instance:FindFirstChildWhichIsA("ProximityPrompt",true)
end

function FruitData:CanPickup(): boolean
    return self:HasGrown() and self:IsWithinPickupRange()
end

function FruitData:IsWithinPickupRange(): boolean
    local pickupPrompt = self:GetPickupPrompt()
    return (pickupPrompt.Parent.Position-getHRP().Position).Magnitude < pickupPrompt.MaxActivationDistance
end

function FruitData:GetMutations(): {string}
    local mutations:{string} = {}
    for _,mutation:Mutation in GAG:GetAllMutations() do
        if self._instance:GetAttribute(mutation.Name) then
            table.insert(mutations,mutation.Name)
        end
    end
    return mutations
end

function FruitData.new(fruitInstance:Instance): FruitData
    local self:FruitData = setmetatable({}, FruitData)

    self._Age = fruitInstance.Grow.Age
    self._ItemSeed = fruitInstance.Item_Seed
    self._Variant = fruitInstance.Variant
    self._Weight = fruitInstance.Weight
    self._instance = fruitInstance

    return self
end

function GAG:GetOwnFarmFolder(): Folder
    for _,farm in workspace.Farm:GetChildren() do
        if farm.Important.Data.Owner.Value == plr.Name then
            return farm
        end
    end
end

function GAG:BuySeed(seedName:string): nil
    gameEvents.BuySeedStock:FireServer(seedName)
end

function GAG:SellFruitInHand(): nil
    gameEvents.Sell_Item:FireServer()
end

function GAG:SellAllFruitsInInventory(): nil
    gameEvents.Sell_Inventory:FireServer()
end

function GAG:PlantSeed(seedName:string, plantPosition:Vector3): nil
    gameEvents.Plant_RE:FireServer(plantPosition, seedName)
end

function GAG:GetLastSeedShopResetTimestamp(): number
    return dataService:GetData().SeedStock.ForcedSeedEndTimestamp
end

function GAG:GetSeedShopStock(): StockObject
    return dataService:GetData().SeedStock.Stocks
end

function GAG:GetLastPetEggShopResetTimestamp(): number
    return dataService:GetData().PetEggStock.ForcedEggEndTimestamp
end

function GAG:IsInventoryFull(): boolean
    return inventoryService:IsMaxInventory()
end

type ItemData = { -- generalized itemdata type
    ItemName:string,
    IsFavorite:boolean,
    Seed:number,
    MutationString:string, -- moonlit, wet
    Variant:string, -- gold, rainbow
    WeightMultiplier:number,
    Quantity:number, -- not present in plants btw
    Weight:number
}

type InventoryClassName = "PetEgg" | "Seed" | "Holdable"

type InventoryItem = { -- generalized inventoryitem class
    ItemData:ItemData,
    ItemType:string,
    UUID:string, -- WITH BRACKETS
    EggName: string, -- eggs
    Uses: number, -- eggs
    GetClass: (self:InventoryItem) -> InventoryClassName,
    IsSeedOrFruit: (self:InventoryItem) -> boolean,
    IsSeed: (self:InventoryItem) -> boolean,
    IsFruit: (self:InventoryItem) -> boolean,
    GetTool: (self:InventoryItem) -> Tool
}

local InventoryItem:InventoryItem = {}
InventoryItem.__index = InventoryItem

function InventoryItem:GetTool(): Tool
    for i,v in plr.Backpack:GetChildren() do
        if v:GetAttribute("ITEM_UUID") == self.UUID or v:GetAttribute("PET_UUID") == self.UUID then
            return v
        end
    end
    local charTool = plr.Character:FindFirstChildWhichIsA("Tool")
    if charTool and charTool:GetAttribute("ITEM_UUID") == self.UUID and charTool:GetAttribute("PET_UUID") == self.UUID then
        return charTool
    end
end

function InventoryItem.new(dict:table): InventoryItem
    return setmetatable(dict,InventoryItem)
end

function InventoryItem.fromInventory(itemUUID:string): InventoryItem
    local data = dataService:GetData()
    local item = data.InventoryData[itemUUID] or data.PetsData.PetInventory.Data[itemUUID]
    item.UUID = itemUUID
    return InventoryItem.new(item)
end

function InventoryItem.fromTool(tool:Tool): InventoryItem
    local uuidValueInstance = tool:FindFirstChild("ITEM_UUID") or tool:FindFirstChild("PET_UUID")
    print("got uuid", uuidValueInstance.Value)

    if uuidValueInstance then
        dprint("returning from inventory")
        return InventoryItem.fromInventory("{"..uuidValueInstance.Value.."}")
    end

    -- for trading
    -- mappings help translate tool children properties and attributes into an inventoryitem

    local toolAttributeStringMappings = {
        ["Favorite"] = "IsFavorite",
        ["ITEM_TYPE"] = "ItemType",
        ["ITEM_UUID"] = "UUID",
        ["ItemName"] = "ItemName",
        ["ItemType"] = "ItemType",
        ["WeightMulti"] = "WeightMultiplier",
        ["Quantity"] = "Quantity",
        ["PET_UUID"] = "UUID"
    }

    local toolValueInstanceNameMappings = {
        ["Item_Seed"] = "Seed",
        ["Item_String"] = "ItemName",
        ["Variant"] = "Variant",
        ["Weight"] = "Weight"
    }

    local mutationStrings = {}

    for i,v in GAG:GetAllMutations() do
        if tool:GetAttribute(v.Name) then
            table.insert(mutationStrings,v.Name)
        end
    end

    local inventoryItem: InventoryItem = {}
    local isPet = tool:GetAttribute("PET_UUID") ~= nil
    local dataPath

    if isPet then
        inventoryItem.PetData = {}
        dataPath = inventoryItem.PetData
    else
        inventoryItem.ItemData = {}
        dataPath = inventoryService.ItemData
    end

    if #mutationStrings > 0 then
        dataPath.MutationString = table.concat(mutationStrings,", ")
    end

    for attribute, mapping in toolAttributeStringMappings do
        dataPath[mapping] = tool:GetAttribute(attribute)
    end

    for valueInstanceName, mapping in toolValueInstanceNameMappings do
        local valueInstance = tool:FindFirstChild(valueInstanceName)
        dataPath[mapping] = valueInstance and valueInstance.Value
    end

    inventoryItem.UUID = dataPath.UUID -- to help make the structure more like the games internals item structure
    inventoryItem.ItemType = dataPath.ItemType -- same as above

    return InventoryItem.new(inventoryItem)
end

function InventoryItem:IsSeedOrFruit(): boolean
    if self:GetClass() ~= "Holdable" and self:GetClass() ~= "Seed" then
        return false
    end
    return seedData[self.ItemData.ItemName] and true
end

function GAG:IsSeedOrFruit(itemName:string): boolean
    return seedData[itemName] and true
end

function InventoryItem:GetClass(): "Seed" | "Holdable" | "Pet"
    return self.ItemType
end

function InventoryItem:IsSeed(): boolean
    return self:GetClass() == "Seed"
end

function InventoryItem:IsFruit(): boolean
    return self:IsSeedOrFruit() and (not self:IsSeed()) and true
end

function GAG:NightQuestSubmitHeldPlant(): nil
    local ohString1 = "SubmitHeldPlant"
    gameEvents.NightQuestRemoteEvent:FireServer(ohString1)
end

function GAG:NightQuestSubmitAllPlants(): nil
    local ohString1 = "SubmitAllPlants"
    gameEvents.NightQuestRemoteEvent:FireServer(ohString1)
end

function GAG:GetNextGlobalUpdateTimestamp(): number
    return rf.GlobalUpdateTime.Value
end

function GAG:GetCurrentBiome(): string
    return plr.Current_Biome.Value
end

local NightEventShopItems = require(rs.Data.NightEventShopData)

function GAG:BuyNightEventShopItem(itemName:string): nil
    local ohString1 = itemName
    gameEvents.BuyNightEventShopStock:FireServer(ohString1)
end

type StockObject = {
    ItemName:string,
    Stock:number,
    MaxStock:number -- not always present
}

function GAG:GetPetEggsStock(): {StockObject}
    local eggs = {}
    for _,eggTable in dataService:GetData().PetEggStock.Stocks do
        table.insert(eggs,{
            ItemName = eggTable.EggName,
            Stock = eggTable.Stock,
            MaxStock = nil -- no max stock for eggs, wont be using that like ever anyway so
        })
    end
    return eggs
end

function GAG:GetNightEventShopStock(): {StockObject}
    local stocks = {}
    for itemName, stock in dataService:GetData().NightEventShopStock.Stocks do
        table.insert(stocks,{ItemName=itemName,Stock=stock.Stock,MaxStock=stock.MaxStock})
    end
    return stocks
end

function GAG:GetEventShopStock(): {StockObject}
    local stocks = {}
    for itemName, stock in dataService:GetData().EventShopStock.Stocks do
        table.insert(stocks,{ItemName=itemName,Stock=stock.Stock,MaxStock=stock.MaxStock})
    end
    return stocks
end

function GAG:GetGearStock(): {StockObject}
    local stocks = {}
    for itemName, stock in dataService:GetData().GearStock.Stocks do
        table.insert(stocks,{ItemName=itemName,Stock=stock.Stock,MaxStock=stock.MaxStock})
    end
    return stocks
end

type CrateStockObject = {
    Stock:number,
    CrateName:string,
    ItemName:string -- artificial (key added from dict)
}

type CosmeticItemStockObject = {
    Stock:number,
    ItemName:string -- artificial (key added from dict)
}

type CosmeticStocks = {
    CrateStocks: {CrateStockObject},
    ItemStocks: {CosmeticItemStockObject}
}

function GAG:GetCosmeticStock(): {CosmeticStocks}
    local CrateStocks = {}
    local ItemStocks = {}
    for stocksObjectName, stockObjects in dataService:GetData().CosmeticStock do
        if stocksObjectName == "CrateStocks" then
            for i,v:CrateStockObject in stockObjects do
                v.ItemName = v.CrateName
                table.insert(CrateStocks,v)
            end
        elseif stocksObjectName == "ItemStocks" then
            for i,v:CosmeticItemStockObject in stockObjects do
                v.ItemName = i
                table.insert(ItemStocks,v)
            end
        end
    end
    return {
        CrateStocks = CrateStocks,
        ItemStocks = ItemStocks
    }
end

function GAG:CalculateFruitValueMultiplierFromMutations(fruit:InventoryItem): number
    local v294 = 1;
    if fruit.MutationString ~= "" then
        for _, mutation in GAG:GetAllMutations() do
            if fruit.MutationString:find(mutation.Name) then
                v294 = v294 + (mutation.ValueMulti - 1);
            end
        end
    end
    return (math.max(1, v294));
end

type LocationCFrames = {
    SellCFrame: CFrame,
    SeedsShopCFrame: CFrame,
    GardenCFrame: CFrame,
    GearShopCFrame: CFrame,
    EventCFrame: CFrame,
    PetEggShopCFrame: CFrame
}

local LocationCFrames:LocationCFrames = { -- add logic for getting different shop, sell, event cframes etc if there is a new world or sum
    SellCFrame = CFrame.new(86.5854721, 2.76619363, 0.426784277, 0, 0, -1, 0, 1, 0, 1, 0, 0),
    SeedsShopCFrame = CFrame.new(86.5854721, 2.76619363, -27.0039806, 0, 0, -1, 0, 1, 0, 1, 0, 0),
    GearShopCFrame = CFrame.new(-284.41452, 2.76619363, -32.9778976, 0, 0, 1, 0, 1, -0, -1, 0, 0),
    EventCFrame = CFrame.new(-100.685043, 0.90001297, -11.6456203, 0, 1, 0, 0, 0, 1, 1, 0, 0),
    PetEggShopCFrame = CFrame.new(-285.279419, 2.99999976, -32.4928207, 0.0324875265, -9.37526856e-09, 0.999472141, 1.13912598e-07, 1, 5.67752734e-09, -0.999472141, 1.13668023e-07, 0.0324875265)
}

setmetatable(LocationCFrames,{
    __index = function(t,k)
        if k == "GardenCFrame" then
            return GAG:GetOwnFarmFolder().Spawn_Point.CFrame -- in case own farm folder ever changes for some reason
        end
    end
})

local function TPLocation(locationCFrame:CFrame)
    plr.Character.HumanoidRootPart.CFrame = locationCFrame
end

function GAG:IsBloodMoonActive(): boolean
    return workspace:GetAttribute("BloodMoonEvent")
end

function GAG:GetEquippedPetUUIDs(): {string}
    return dataService:GetData().PetsData.EquippedPets
end

function GAG:GetPetUUIDsInInventory(): {string}
    local uuids = {}
    for uuid, _ in dataService:GetData().PetsData.PetInventory.Data do
        table.insert(uuids,uuid)
    end
    return uuids
end

type PetData = {
    BaseWeight: number,
    Level: number,
    Hunger: number,
    Name: string,
    HatchedFrom: string,
    LevelProgress: number
}

type PetInventoryItem = {
    PetType: string,
    PetData: PetData,
    PetAbility: {string},
    UUID: string, -- with brackets -- artificial (key added from dict)
    EquipPet: (self:PetInventoryItem) -> nil,
    UnequipPet: (self:PetInventoryItem) -> nil,
    FeedPetItemFromHand: (self:PetInventoryItem) -> nil,
}

local PetInventoryItem:PetInventoryItem = {}
PetInventoryItem.__index = PetInventoryItem

function PetInventoryItem.fromInventory(petItemData:table): PetInventoryItem
    return setmetatable(petItemData,PetInventoryItem)
end

function PetInventoryItem:EquipPet(spawnCFrame:CFrame): nil -- cframe has to be within your farm
    local ohString1 = "EquipPet"
    local ohString2 = self.UUID -- WITH BRACKETS TODO
    local ohCFrame3 = spawnCFrame
    gameEvents.PetsService:FireServer(ohString1, ohString2, ohCFrame3)
end

function PetInventoryItem:UnequipPet(): nil
    local ohString1 = "UnequipPet"
    local ohString2 = self.UUID -- WITH BRACKETS
    gameEvents.PetsService:FireServer(ohString1, ohString2)
end

function PetInventoryItem:FeedPetItemFromHand(): nil
    local ohString1 = "Feed"
    local ohString2 = self.UUID -- WITH BRACKETS
    gameEvents.ActivePetService:FireServer(ohString1, ohString2)
end

function GAG:GetPetByUUID(petUUID:string): PetInventoryItem
    local petInventoryItem = dataService:GetData().PetsData.PetInventory.Data[petUUID]
    petInventoryItem.UUID = petUUID
    return PetInventoryItem.fromInventory(petInventoryItem)
end

function GAG:GetEquippedPets(): {PetInventoryItem}
    local equippedUUIDs = GAG:GetEquippedPetUUIDs()
    local petInventoryItems: {PetInventoryItem} = {}
    for _,uuid in equippedUUIDs do
        table.insert(petInventoryItems,GAG:GetPetByUUID(uuid))
    end
    return petInventoryItems
end

function GAG:GetPetsInInventory(): {PetInventoryItem}
    local petUUIDs = GAG:GetPetUUIDsInInventory()
    local petInventoryItems: {PetInventoryItem} = {}
    for _,uuid in petUUIDs do
        table.insert(petInventoryItems,GAG:GetPetByUUID(uuid))
    end
    return petInventoryItems
end

function GAG:BuyEventShopItem(itemName:string): nil
    gameEvents.BuyEventShopStock:FireServer(itemName)
end

function GAG:IsNightEventActive(): boolean
    return workspace:GetAttribute("NightEvent")
end


function GAG:GetInventory(): {InventoryItem}
    local items:{InventoryItem} = {}
    for uuid, itemTable in dataService:GetData().InventoryData do
        local inventoryItem = itemTable
        inventoryItem.UUID = uuid
        table.insert(items,
            InventoryItem.new(inventoryItem)
        )
    end
    return items
end

function GAG:GetAllMutations(): {Mutation}
    local MutationHandler = require(rs.Modules.MutationHandler)
    local mutations:{Mutation} = {}
    for _,mutation in MutationHandler:GetMutations() do
        table.insert(mutations,mutation)
    end
    return mutations
end

function GAG:GetShecklesCurrency(): number
    return dataService:GetData().Sheckles or 0
end

type PetEggItem = {
    EggName:string,
    Uses:number
}

type SeedItem = {
    Quantity: number,
    ItemName: string,
    Variant: string
}

function GAG:PlantEggInHand(eggPosition:Vector3): nil
    local ohString1 = "CreateEgg"
    local ohVector32 = Vector3.new(eggPosition.X,0.13552704453468323,eggPosition.Z)
    gameEvents.PetEggService:FireServer(ohString1, ohVector32)
end

type SavedObject = {
    ObjectType:string,
    RelativeCFrame:table, -- unpack to construct into cframe
    UUID:string
}

type GeneratedPetData = {
    WeightRange: {number},
    HugeChance:number
}

type RandomPetData = {
    NormalizedOdd: number,
    Name: string,
    ItemOdd: number
}

type PetEggObjectData = {
    CanOpen:boolean,
    TimeToHatch:number,
    EggName:string,
    RandomPetData: RandomPetData,
    Type: string,
    TimeToHatch:number,
    CanOpen:boolean,
    CanHatch:boolean,
    BaseWeight: number
}

type PetEggObject = SavedObject & {
    Data: PetEggObjectData,
    IsPredictable: (self:PetEggObject) -> boolean,
}

local PetEggObject:PetEggObject = {}
PetEggObject.__index = PetEggObject

function PetEggObject.new(dict:table): PetEggObject
    return setmetatable(dict,PetEggObject)
end

function PetEggObject:IsPredictable(): boolean
    return self.Data.RandomPetData and self.Data.RandomPetData.Name and true
end

function GAG:GetSavedObjects(): {SavedObject} -- mainly Eggs, plants wont show here
    local objects = {}
    for uuid, objectTable in dataService:GetData().SavedObjects do
        local object = objectTable
        object.UUID = uuid
        table.insert(objects,object)
    end
    return objects
end

function GAG:GetPlantedEggObjects() : {PetEggObject}
    local eggs = {}
    for _,object:PetEggObject in GAG:GetSavedObjects() do
        if object.ObjectType ~= "PetEgg" then
            continue
        end
        table.insert(eggs,PetEggObject.new(object))
    end
    return eggs
end

function GAG:HatchEgg(eggInstance:Instance): nil -- .PetEgg Model, no distance check
    local ohString1 = "HatchPet"
    local ohInstance2 = eggInstance
    gameEvents.PetEggService:FireServer(ohString1, ohInstance2)
end

function GAG:BuyEggFromPetEggShop(eggIndex:number): nil
    local ohNumber1 = eggIndex
    gameEvents.BuyPetEgg:FireServer(ohNumber1)
end

local petRegistry = require(rs.Data.PetRegistry)

function GAG:GetPetMaxHunger(petName:string): number -- hunger doesnt scale yet so return DefaultHunger for every pet
    return petRegistry.PetList[petName].DefaultHunger
end

function GAG:GetPlantsOnFarm(): {PlantData}
    local plantsData:{PlantData} = {}
    for _,plantInstance:Instance in GAG:GetOwnFarmFolder().Important.Plants_Physical:GetChildren() do
        table.insert(plantsData,PlantData.new(plantInstance))
    end
    return plantsData
end

function GAG:GetFruitsOnFarm(): {FruitData}
    local fruits:{FruitData} = {}
    for _,plantData in GAG:GetPlantsOnFarm() do
        for _,fruitData in plantData:GetFruitsData() do
            table.insert(fruits,fruitData)
        end
    end
    return fruits
end

function GAG:GetMaxEquippedPets(): number
    return dataService:GetData().PetsData.MutableStats.MaxEquippedPets
end

function GAG:GetMaxPetsInInventory(): number
    return dataService:GetData().PetsData.MutableStats.MaxPetsInInventory
end

function GAG:GetMaxEggsInFarm(): number
    return dataService:GetData().PetsData.MutableStats.MaxEggsInFarm
end

type ItemClassName = "Pet" | "Holdable" | "Seed" | "Egg"

type AbstractItem = { -- inventory item or pet inventory item
    _toolInstance:Tool,
    GetClass: (self:AbstractItem) -> ItemClassName,
    GetTool: (self:AbstractItem) -> Tool
}

local AbstractItem:AbstractItem = {}
AbstractItem.__index = AbstractItem

function AbstractItem.new(abstractItem:table): AbstractItem
    return setmetatable(abstractItem,AbstractItem)
end

function AbstractItem:GetClass(): ItemClassName
    return self._toolInstance:GetAttribute("ItemType")
end

function AbstractItem:GetTool(): Tool
    return self._toolInstance
end

function GAG:ShovelPlant(instance:Instance): nil -- FruitInstance.PrimaryPart or PlantInstance.Base
    local ohInstance1 = instance
    gameEvents.Remove_Item:FireServer(ohInstance1)
end

function GAG:GetRandomPlantingLocation(): Vector3
    local plantLocations = GAG:GetOwnFarmFolder().Important.Plant_Locations:GetChildren()
    local plantLocation = plantLocations[math.random(#plantLocations)]
    local position = plantLocation.Position
    local size = plantLocation.Size

    local randomX = position.X + (math.random() - 0.5) * size.X
    local randomZ = position.Z + (math.random() - 0.5) * size.Z

    local randomPosition = Vector3.new(randomX, 0.13552704453468323, randomZ) -- cunty Y planting constant

    return randomPosition
end

type GAGSetting = "Audio" | "RecieveGifts"

function GAG:SetSetting(setting:GAGSetting, boolean:boolean): nil
    local ohString1 = "SetSetting"
    local ohString2 = setting
    local ohBoolean3 = boolean
    gameEvents.SettingsService:FireServer(ohString1, ohString2, ohBoolean3)
end

function GAG:RedeemCode(code:string): nil
    local ohString1 = "ClaimCode"
    local ohString2 = code
    gameEvents.ClaimableCodeService:FireServer(ohString1, ohString2)
end

function GAG:GetSpecialCurrency(currencyName:string): number
    local specialCurrency = dataService:GetData().SpecialCurrency
    if (not specialCurrency) or (not specialCurrency[currencyName])  then
        return -1
    end
    return specialCurrency[currencyName]
end

function GAG:GetHoneyCurrency(): number
    return GAG:GetSpecialCurrency("Honey")
end

function GAG:GetFruitsInInventory(): {InventoryItem}
    local fruits:{InventoryItem} = {}
    for _, item in GAG:GetInventory() do
        if GAG:IsFruit(item) then
            table.insert(fruits,item)
        end
    end
    return fruits
end

local MAX_HONEY_MACHINE_PLANT_WEIGHT = 10

type HoneyMachineState = {
    PlantWeight:number,
    HoneyStored:number,
    IsRunning:boolean,
    TimeLeft:number,
    SubmittedPlants:{InventoryItem}
}

function GAG:GetHoneyMachineState(): HoneyMachineState
    return dataService:GetData().HoneyMachine
end

function GAG:IsHoneyMachineFull(): boolean
    return GAG:GetHoneyMachineState().PlantWeight >= MAX_HONEY_MACHINE_PLANT_WEIGHT
end

function GAG:GivePlantToHoneyMachineFromHand(): nil -- no distance check
    local ohString1 = "MachineInteract"
    gameEvents.HoneyMachineService_RE:FireServer(ohString1)
end

function GAG:BuyCosmeticItem(itemName:string): nil
    gameEvents.BuyCosmeticItem:FireServer(itemName)
end

function GAG:BuyCosmeticCrate(crateName:string): nil
    gameEvents.BuyCosmeticCrate:FireServer(crateName)
end

function GAG:ScrapeDevProducts(): table
    local marketplaceService = cloneref(game:GetService("MarketplaceService"))
    local pages:Pages = marketplaceService:GetDeveloperProductsAsync() -- fuck is that but it works
    local items = {}
	while true do
		table.insert(items, pages:GetCurrentPage())
		if pages.IsFinished then
			break
		end
		pages:AdvanceToNextPageAsync()
	end
	return items
end

function GAG:ToggleFavoriteItem(tool:Tool): nil
    local ohInstance1 = tool
    gameEvents.Favorite_Item:FireServer(ohInstance1)
end

function GAG:ScrapeAllWeathers(): table
    return httpService:JSONDecode(workspace:GetAttribute("AllWeather"))
end

type CollectableSeed = {
    Name: string,
    OWNER: string, -- player name
    _instance: Instance,
    GetTouchInterest: (self: CollectableSeed) -> TouchTransmitter
}

local CollectableSeed:CollectableSeed = {}
CollectableSeed.__index = CollectableSeed

function CollectableSeed:GetTouchInterest(): TouchTransmitter
    return self._instance.PrimaryPart:FindFirstChildWhichIsA("TouchTransmitter")
end

function CollectableSeed.new(dict: table): CollectableSeed
    local self = setmetatable(dict, CollectableSeed)
    return self
end

function GAG:GetCollectableSeeds(): {CollectableSeed}
    local collectableSeeds: {CollectableSeed} = {}

    for _, v in workspace:GetChildren() do
        if v:IsA("Model") then
            local owner = v:GetAttribute("OWNER")
            local seedGiven = v:GetAttribute("SEED_GIVEN")
            if owner and seedGiven then
                table.insert(collectableSeeds, CollectableSeed.new({
                    Name = seedGiven,
                    OWNER = owner,
                    _instance = v
                }))
            end
        end
    end

    return collectableSeeds
end

function GAG:GivePetInHand(targetPlayer:Player): nil
    local ohString1 = "GivePet"
    local ohInstance2 = targetPlayer
    gameEvents.PetGiftingService:FireServer(ohString1, ohInstance2)
end

local autoPetGiftsConnection

function GAG:AutoAcceptPetGifts(enable:boolean) -- way cleaner way if u only want pet gifts
    if enable then
        autoPetGiftsConnection = gameEvents.GiftPet.OnClientEvent:Connect(function(uuid, fullPetName, offeringPlayer) -- UUID HAS NO BRACKETS
            --print(uuid, fullPetName, offeringPlayer)
            gameEvents.AcceptPetGift:FireServer(true, uuid)
            -- decline with
            -- game.ReplicatedStorage.GameEvents.AcceptPetGift:FireServer(false, uuid)
        end)
    elseif autoPetGiftsConnection then
        autoPetGiftsConnection:Disconnect()
        autoPetGiftsConnection = nil
    end
end

function GAG:RemoveBracketsFromUUID(uuid:string): string
    local cleaned_str = uuid:sub(2, -2)
    return cleaned_str
end

function GAG:GiftItemInHand(targetPlayer:Player): nil
    fireproximityprompt(targetPlayer.Character:FindFirstChildWhichIsA("ProximityPrompt",true))
end

function GAG:LikeGarden(player:Player) -- no distance check
    local ohInstance1 = player
    gameEvents.LikeGarden:InvokeServer(ohInstance1)
end

function GAG:GetPetEggInstanceByUUID(uuid:string): Instance
    for i,eggInstance in GAG:GetOwnFarmFolder().Important.Objects_Physical:GetChildren() do
        if eggInstance:GetAttribute("OBJECT_TYPE") == "PetEgg" and eggInstance:GetAttribute("OBJECT_UUID") == uuid then
            return eggInstance
        end
    end
end

function GAG:RejoinServer(): nil -- may rejoin a different server sometimes
    local tpService = game:GetService("TeleportService")
    tpService.TeleportInitFailed:Connect(function(player, teleportResult, errorMessage, placeId, teleportOptions)
        tpService:Teleport(game.PlaceId)
    end)
    tpService:Teleport(game.PlaceId)
end

function GAG:GetToolInHand(): Tool
    return plr.Character:FindFirstChildWhichIsA("Tool")
end

return GAG
