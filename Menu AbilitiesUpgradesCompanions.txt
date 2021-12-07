///////////////////////////////
/*Menu AbilitiesUpgradesCompanions

>-- WHAT / WHY --<
This script implements a system, that displays a custom menu, which allows the main character (a.k.a player) to replace abilities at any given time.
Furthermore this custom menu allows a player to upgrade abilities at the cost of money, buy unique upgrades and companions.
Some of these features already exist in Warcraft 3, but this custom menu presents a solution to unify these features into a single custom menu.
The player has to click less, ergo the user experience improves!


>-- LIMITS --<
The intended abilities, upgrades and companions must be hardcoded into this system, as Warcraft 3 lacks an API that can read some attributes.
Due to the hardcoded nature of some elements, this system was not intended to be re-usable outside the project "Lord of the Clans".
Significant changes would have to be made to this code base and the Warcraft 3 project to make it appropriate for other projects.


>-- HOW IT WAS CREATED --<
This system was created in "vertical slices" in the following order (summarized):
1) Display/hide menu
2) Display a single button and make it clickable (so it can trigger events)
3) Add more abilities; make abilities learnable through the custom menu
4) Add more abilities; make abilities replaceable
5) Make abilities upgradeable
6) Add upgrades; make upgrades researchable
7) Add companions; make companions buyable
8) add support for map-to-map-loading (so the system remembers everything on the next map)
9) fixed game reloading (system used to break when reloading a map)
10) polishing/balancing

>-- HOW TO READ --<
You can download a JASS syntax definition for SublimeText from https://github.com/Ruk33/SublimeText-JASS
Just copy the JASS.subline-package file into your "Sublime Text 3\Packages" folder.

JASS is the propriatery event-driven scripting language behind Warcraft 3, created by Blizzard Entertainment.
JASS follows a prodedural approach, and each function/variable must be declared before it can be used.
The function "InitMenu" initializes the menu and logic and is called when starting or loading a map.
The function "OpenOrCloseMenu" is called when a player casts a custom placeholder ability. It is used display/hide the custom menu.
Other important functions and variables will be explained further down.

>-- DEMONSTRATION --<
I explain and showcase this system: https://youtu.be/TKdVbxCge70

*/


// Global variables that identify the general location of the main menu and each individual item of the menu.
globals
	// MENU
	framehandle background
	boolean menuIsVisible
	real gPosX = 0.54
	real gPosY = 0.36

	framehandle abilitiesText
	framehandle qText
	framehandle wText
	framehandle eText
	framehandle rText
	framehandle upgradesText
	framehandle companionsText

	framehandle closeBackdrop
	framehandle closeHover

	// COMBAT ABILITIES
	//Data
	integer array abilityCode[12]
	integer array abilityUpgrade[12]
	integer array abilitySuperMaxLevel[12]
	integer array abilityMaxLevel[12]
	integer array abilityCurrentLevel[12]
	boolean array abilityActive[12]

	//Button
	real array abilityPosX[12]
	real array abilityPosY[12]
	framehandle array abilityBackdrop[12]
	framehandle array abilityButton[12]
	string array abilityIcon[12]
	string array abilityIconName[12]
	framehandle array abilityCheckbox[12]

	//Tooltip
	framehandle array tooltip[12]
	framehandle array abilityTooltipTitleFh[12]
	string array abilityTooltipTitleText[12]
	framehandle array abilityTooltipBodyFh[12]
	string array abilityTooltipBodyText[12]
	
	
	// IMPROVE COMBAT ABILITIES
	//Button
	framehandle array abilityImproveButton[12]
	framehandle array abilityImproveFh[12]

	//Tooltip
	framehandle array abilityImproveTooltip[12]
	framehandle array abilityImproveTooltipTitleFh[12]
	framehandle array abilityImproveTooltipGoldIcon[12]
	framehandle array abilityImproveTooltipGoldText[12]

	// RESEARCH UPGRADES
	framehandle array researchBackdrop[15]
	framehandle array researchHover[15]

	integer array researchCode[15]
	boolean array researchUnlocked[15]
	boolean array researchApplied[15]
	integer array researchGoldCost[15]
	integer array researchLumberCost[15]

	framehandle array researchTooltip[15]
	string array researchTitle[15]
	string array researchText[15]
	string array researchIcon[15]
	string array researchIconName[15]
	framehandle array researchCheckbox[15]
	
	real array researchPosX[15]
	real array researchPosY[15]

	// COMPANIONS
	framehandle array companionBackdrop[9]
	framehandle array companionHover[9]

	integer array companionCode[9]
	boolean array companionUnlocked[9]
	integer array companionGoldCost[9]
	integer array companionLumberCost[9]
	integer array companionFoodCost[9]

	framehandle array companionTooltip[9]
	string array companionTitle[9]
	string array companionText[9]
	string array companionIcon[9]
	string array companionIconName[9]
	
	real array companionPosX[9]
	real array companionPosY[9]


	
endglobals

////////////////////
// MENU: Loads the background image for the custom menu
function loadTOC takes nothing returns nothing
	call BlzLoadTOCFile("war3mapimported\\BoxedText.toc")
endfunction


// Creates the background and static titles/sub-titles
function CreateBackground takes nothing returns nothing
	local real height = 0.16
	set background = BlzCreateFrame("Background", BlzGetOriginFrame(ORIGIN_FRAME_GAME_UI, 0),0, 0)
   	call BlzFrameSetSize(background, 0.36, 0.40)
   	call BlzFrameSetAbsPoint(background, FRAMEPOINT_CENTER, gPosX, gPosY)

	set abilitiesText = BlzCreateFrame("BackgroundTitle", background,0, 0)
   	call BlzFrameSetAbsPoint(abilitiesText, FRAMEPOINT_CENTER, gPosX, gPosY+0.18)
   	call BlzFrameSetText(abilitiesText, "|cffffcc00Abilities|r")

   	set qText = BlzCreateFrame("BackgroundText", background,0, 0)
   	call BlzFrameSetAbsPoint(qText, FRAMEPOINT_CENTER, gPosX- 0.13, gPosY+height)
   	call BlzFrameSetText(qText, "|cffffcc00Q|r")

   	set wText = BlzCreateFrame("BackgroundText", background,0, 0)
   	call BlzFrameSetAbsPoint(wText, FRAMEPOINT_CENTER, gPosX - 0.05, gPosY+height)
   	call BlzFrameSetText(wText, "|cffffcc00W|r")

   	set eText = BlzCreateFrame("BackgroundText", background,0, 0)
   	call BlzFrameSetAbsPoint(eText, FRAMEPOINT_CENTER, gPosX + 0.03, gPosY+height)
   	call BlzFrameSetText(eText, "|cffffcc00E|r")

   	set rText = BlzCreateFrame("BackgroundText", background,0, 0)
   	call BlzFrameSetAbsPoint(rText, FRAMEPOINT_CENTER, gPosX + 0.11, gPosY+height)
   	call BlzFrameSetText(rText, "|cffffcc00R|r")

   	set upgradesText = BlzCreateFrame("BackgroundTitle", background,0, 0)
   	call BlzFrameSetAbsPoint(upgradesText, FRAMEPOINT_CENTER, gPosX - 0.082, gPosY)
   	call BlzFrameSetText(upgradesText, "|cffffcc00Upgrades|r")

   	set companionsText = BlzCreateFrame("BackgroundTitle", background,0, 0)
   	call BlzFrameSetAbsPoint(companionsText, FRAMEPOINT_CENTER, gPosX + 0.10, gPosY)
   	call BlzFrameSetText(companionsText, "|cffffcc00Companions|r")
	
endfunction


// Open or close menu. Called by using a custom placeholder ability 
function OpenOrCloseMenu takes boolean visi returns nothing
	set menuIsVisible = visi
	call BlzFrameSetVisible(background, visi)
	set udg_MenuIsOpen = visi
endfunction


// Close menu. Mostly used to force closing (i.e. during cinematics)
function CloseButton takes nothing returns nothing
	call OpenOrCloseMenu(false)
endfunction


// Create close button. Creates an icon, and connects it to a click-able element that calls a specific event
function CreateCloseButton takes nothing returns nothing
	local trigger trig = CreateTrigger()

	//Backdrop//Button
	set closeBackdrop = BlzCreateFrameByType("BACKDROP", "Face", background, "", 0)
	call BlzFrameSetSize(closeBackdrop, 0.02, 0.02)
	call BlzFrameSetAbsPoint(closeBackdrop, FRAMEPOINT_CENTER, gPosX+0.165, gPosY-0.185)
	set closeHover = BlzCreateFrameByType("GLUEBUTTON", "FaceFrame", closeBackdrop,"IconButtonTemplate", 0)
	call BlzFrameSetAllPoints(closeHover, closeBackdrop) 

	//Icon
	call BlzFrameSetTexture(closeBackdrop, "ReplaceableTextures\\CommandButtons\\BTNCancel.tga", 0, true)

	//Action
	call BlzTriggerRegisterFrameEvent(trig, closeHover, FRAMEEVENT_CONTROL_CLICK)
	
	call TriggerAddAction(trig, function CloseButton)
endfunction


// Creates an icon for a tooltip, based on the ability that used it
function CreateTooltipIcon takes string name, framehandle owner, framehandle parent, framepointtype relativePoint, real x, real y, string icon returns framehandle
	local framehandle tooltipIcon = BlzCreateFrameByType("BACKDROP", name, owner, "", 0)
	call BlzFrameSetPoint(tooltipIcon, FRAMEPOINT_LEFT, parent, relativePoint, x, y)
	call BlzFrameSetSize(tooltipIcon, 0.009, 0.009)
	call BlzFrameSetTexture(tooltipIcon, "UI\\Widgets\\ToolTips\\Human\\" + icon, 0, true)
	return tooltipIcon
endfunction

// Creates an number for a tooltip, based on the ability that used it (used to display cost)
function CreateTooltipAmount takes string name, framehandle owner, framehandle parent, integer amount returns framehandle
	local framehandle tooltipAmount = BlzCreateFrameByType("TEXT", name, owner, "", 0)
	call BlzFrameSetPoint(tooltipAmount, FRAMEPOINT_LEFT, parent, FRAMEPOINT_RIGHT, 0.004, 0.0)
	call BlzFrameSetText(tooltipAmount, "|cffffcc00" + I2S(amount) + "|r")
	return tooltipAmount
endfunction


////////////////////
// COMBAT ABILITIES: Sets the current and maximum levels for each ability. Also sets tooltips.
function SetCombatAbility takes integer id, integer abil, integer upgr, string btn, integer superMaxLevel returns nothing
	local integer level = GetPlayerTechCountSimple(upgr, Player(0))
	
	set abilityCode[id] = abil
	set abilityIconName[id] = btn
	set abilitySuperMaxLevel[id] = superMaxLevel

	set abilityUpgrade[id] = upgr
	set abilityCurrentLevel[id] = level

	//not researched levels must not go beyond level 1
	if level == 0 then
		set level = 1
	endif
	set abilityTooltipTitleText[id] = BlzGetAbilityTooltip(abil, level -1) //start count at 0
	set abilityTooltipBodyText[id] = BlzGetAbilityExtendedTooltip(abil, level -1) //start count at 0
	set abilityMaxLevel[id] = GetPlayerTechMaxAllowedSwap(upgr, Player(0))

	if (abilityCurrentLevel[id] > 0) then
		set abilityIcon[id] = BlzGetAbilityIcon(abil)
	else
		set abilityIcon[id] = "ReplaceableTextures\\CommandButtonsDisabled\\DIS" + btn
	endif
endfunction


//Decides which abilities are part of this system
function InitCombatAbilities takes nothing returns nothing
	local real diffX = 0.080
	local real diffY = 0.045
	set menuIsVisible = false
	
	// Q
	//Knockback
	call SetCombatAbility(1, 'A63H', 'R60R', "BTNCourage.blp", 5)
	set abilityPosX[1] = gPosX - 0.13
	set abilityPosY[1] = gPosY + 0.13

	//Earthshock
	call  SetCombatAbility(2, 'A63I', 'R60D', "BTNBash.blp", 5)
	set abilityPosX[2] = abilityPosX[1]
	set abilityPosY[2] = abilityPosY[1] - diffY

	//Chain Lightning
	call SetCombatAbility(3, 'A62F', 'R60H', "BTNChainLightning.blp", 5)
	set abilityPosX[3] = abilityPosX[1]
	set abilityPosY[3] = abilityPosY[2] - diffY

	// W
	//Raging Strike
	call SetCombatAbility(4, 'A63O', 'R60A', "BTNRagingstrike.blp", 5)
	set abilityPosX[4] = abilityPosX[1] + diffX
	set abilityPosY[4] = abilityPosY[1]

	//Whirlwind
	call SetCombatAbility(5, 'A64B', 'R610', "BTNHot Wirlwind.blp", 5)
	set abilityPosX[5] = abilityPosX[4]
	set abilityPosY[5] = abilityPosY[2]

	//Ice Shards
	call SetCombatAbility(6, 'A66C', 'R612', "BTNIceBolt.blp", 5)
	set abilityPosX[6] = abilityPosX[4]
	set abilityPosY[6] = abilityPosY[3]

	// E
	//Ravage
	call SetCombatAbility(7, 'A63S', 'R611', "BTNAbility_Warrior_Challange.blp", 5)
	set abilityPosX[7] = abilityPosX[4] + diffX
	set abilityPosY[7] = abilityPosY[1]

	//Duel
	call SetCombatAbility(8, 'A660', 'R60F', "BTNRavage.blp", 5)
	set abilityPosX[8] = abilityPosX[7]
	set abilityPosY[8] = abilityPosY[2]

	//Summon Fire Elementals
	call SetCombatAbility(9, 'A63F', 'R60I', "BTNFireSpawn.blp", 5)
	set abilityPosX[9] = abilityPosX[7]
	set abilityPosY[9] = abilityPosY[3]

	// R
	//Indestructible
	call SetCombatAbility(10, 'A61X', 'R60B', "BTNMuscleCharge.blp", 3)
	set abilityPosX[10] = abilityPosX[7] + diffX
	set abilityPosY[10] = abilityPosY[1]

	//Challenge
	call SetCombatAbility(11, 'A63Z', 'R60S', "BTNTuskarrBanner.blp", 3)
	set abilityPosX[11] = abilityPosX[10]
	set abilityPosY[11] = abilityPosY[2]

	//Will of the Wilds
	call SetCombatAbility(12, 'A649', 'R60T', "BTNChickenCritter.blp", 3)
	set abilityPosX[12] = abilityPosX[10]
	set abilityPosY[12] = abilityPosY[3]

endfunction

//Update ability tooltip, used when an ability has been upgraded
function UpdateAbilityTooltip takes integer id returns nothing
	call SetCombatAbility(id, abilityCode[id], abilityUpgrade[id], abilityIconName[id], abilitySuperMaxLevel[id])
	call BlzFrameSetTexture(abilityBackdrop[id], abilityIcon[id], 0, true)
	call BlzFrameSetText(abilityTooltipTitleFh[id], abilityTooltipTitleText[id])
	call BlzFrameSetText(abilityTooltipBodyFh[id], abilityTooltipBodyText[id])
endfunction


//Creates a checkbox to identify which ability is currently in use
function CreateAbilityCheckbox takes integer id, boolean visi returns nothing
	set abilityCheckbox[id] = BlzCreateFrameByType("BACKDROP", "Face", background, "", 0)
	call BlzFrameSetSize(abilityCheckbox[id], 0.02, 0.02)
	call BlzFrameSetAbsPoint(abilityCheckbox[id], FRAMEPOINT_CENTER, abilityPosX[id] + 0.015, abilityPosY[id] -0.015)
	call BlzFrameSetTexture(abilityCheckbox[id], "UI\\Widgets\\EscMenu\\Human\\checkbox-check.blp", 0, true)
	call BlzFrameSetVisible(abilityCheckbox[id], visi)
endfunction


//Checks if there are enemies near the player. Can't open the menu if enemies are nearby
function FilterNearbyEnemies takes nothing returns boolean
    return ( IsUnitAliveBJ(GetFilterUnit()) == true) and (IsUnitEnemy(GetFilterUnit(), GetOwningPlayer(udg_Thrall)) == true )
endfunction


//Each ability belongs to a category labelled "Q", "W", "E", "R" (corresponding to keyboard hotkeys)
//This function disabled the currently active ability for the selected category and activates the newly selected ability
function ActivateAbility takes integer act, integer deact1, integer deact2, string hotkey, string abilityName returns nothing
	if (abilityCurrentLevel[act] > 0 and abilityActive[act] == false) then	//otherwise don't react at all
	    if ( CountUnitsInGroup(GetUnitsInRangeOfLocMatching(512.00, GetUnitLoc(udg_Thrall), Condition(function FilterNearbyEnemies))) > 0 ) then
	        call QueuedTriggerAddBJ( gg_trg_Ability_EnemiesNearby_Q, true )
	    else
	    	call BlzFrameSetVisible(abilityCheckbox[act], true)
	    	call BlzFrameSetVisible(abilityCheckbox[deact1], false)
	    	call BlzFrameSetVisible(abilityCheckbox[deact2], false)
	        call SetPlayerAbilityAvailableBJ( true, abilityCode[act], Player(0) )
	        call SetPlayerAbilityAvailableBJ( false, abilityCode[deact1], Player(0) )
	        call SetPlayerAbilityAvailableBJ( false, abilityCode[deact2], Player(0) )
	        set abilityActive[act] = true
	        set abilityActive[deact1] = false
	        set abilityActive[deact2] = false
	        if (hotkey == "Q") then
	        	set udg_AbilityCurrentQ = abilityName
	        elseif (hotkey == "W") then
	        	set udg_AbilityCurrentW = abilityName
	        elseif (hotkey == "E") then
	        	set udg_AbilityCurrentE = abilityName
	        elseif (hotkey == "R") then
	        	set udg_AbilityCurrentR = abilityName
	        endif
	    endif
	endif
endfunction

//Each ability belongs to a category labelled "Q", "W", "E", "R" (corresponding to keyboard hotkeys)
// and a specific number, fron 1 to 3. The first number identifies which ability is to be activated,
// the other two numbers identify which abilities are to be deactivated
function ActivateKnockback takes nothing returns nothing	
	call ActivateAbility(1, 2, 3, "Q", "knockback")
endfunction

function ActivateEarthshock takes nothing returns nothing
	call ActivateAbility(2, 1, 3, "Q", "earthshock")
endfunction

function ActivateChainLightning takes nothing returns nothing
	call ActivateAbility(3, 1, 2, "Q", "chainlightning")
endfunction

function ActivateRagingStrike takes nothing returns nothing
	call ActivateAbility(4, 5, 6, "W", "ragingstrike")
endfunction

function ActivateWhirlwind takes nothing returns nothing
	call ActivateAbility(5, 4, 6, "W", "whirlwind")
endfunction

function ActivateIceShards takes nothing returns nothing
	call ActivateAbility(6, 4, 5, "W", "iceShards")
endfunction

function ActivateDuel takes nothing returns nothing
	call ActivateAbility(7, 8, 9, "E", "duel")
endfunction

function ActivateRavage takes nothing returns nothing
	call ActivateAbility(8, 7, 9, "E", "ravage")
endfunction

function ActivateSummonFireElementals takes nothing returns nothing
	call ActivateAbility(9, 8, 7, "E", "summonfirelementals")
endfunction

function ActivateIndestructible takes nothing returns nothing
	call ActivateAbility(10, 11, 12, "R", "indestructible")
endfunction

function ActivateChallenge takes nothing returns nothing
	call ActivateAbility(11, 10, 12, "R", "challenge")
endfunction

function ActivateWillOfTheWilds takes nothing returns nothing
	call ActivateAbility(12, 10, 11, "R", "willofthewilds")
endfunction



// Create button for an ability and connect it to its corresponding function
function CreateAbilityButton takes integer id returns nothing
	local trigger trig = CreateTrigger()

	//Setup
	call SetCombatAbility(id, abilityCode[id], abilityUpgrade[id], abilityIconName[id], abilitySuperMaxLevel[id])

	//Button
	set abilityButton[id] = BlzCreateFrameByType("GLUEBUTTON", "FaceButton", background,"IconButtonTemplate", 0)
	call BlzFrameSetAbsPoint(abilityButton[id], FRAMEPOINT_CENTER, abilityPosX[id], abilityPosY[id])
	call BlzFrameSetSize(abilityButton[id], 0.04, 0.04)

	//Icon
	set abilityBackdrop[id] = BlzCreateFrameByType("BACKDROP", "FaceButtonIcon", abilityButton[id], "", 0)
	call BlzFrameSetAllPoints(abilityBackdrop[id], abilityButton[id]) 
	call BlzFrameSetTexture(abilityBackdrop[id], abilityIcon[id], 0, true)
	
	//Tooltip
	set tooltip[id] = BlzCreateFrame("BoxedText", abilityBackdrop[id], 0, 0)
	call BlzFrameSetPoint(tooltip[id], FRAMEPOINT_BOTTOM, abilityBackdrop[id], FRAMEPOINT_TOP, 0.0, 0.0)
	call BlzFrameSetSize(tooltip[id], 0.18, 0.08)
	call BlzFrameSetTooltip(abilityButton[id], tooltip[id]) 

	//Tooltip Text
	set abilityTooltipBodyFh[id] = BlzCreateFrameByType("TEXT", "BoxedTextValue", tooltip[id], "BoxedTextValue", 0)
	call BlzFrameSetPoint(abilityTooltipBodyFh[id], FRAMEPOINT_BOTTOM, abilityBackdrop[id], FRAMEPOINT_TOP, 0.0, 0.0)
	call BlzFrameSetSize(abilityTooltipBodyFh[id], 0.169, 0.06)
	call BlzFrameSetText(abilityTooltipBodyFh[id], abilityTooltipBodyText[id])


	set abilityTooltipTitleFh[id] = BlzCreateFrameByType("TEXT", "BoxedTextTitle", tooltip[id], "BoxedTextTitle", 0)
	call BlzFrameSetPoint(abilityTooltipTitleFh[id], FRAMEPOINT_LEFT, abilityTooltipBodyFh[id], FRAMEPOINT_LEFT, 0.0, 0.039)
	call BlzFrameSetText(abilityTooltipTitleFh[id], abilityTooltipTitleText[id])

	//Action
	call BlzTriggerRegisterFrameEvent(trig, abilityButton[id], FRAMEEVENT_CONTROL_CLICK)
	
	if id == 1 then
		call TriggerAddAction(trig, function ActivateKnockback)
	elseif id == 2 then
		call TriggerAddAction(trig, function ActivateEarthshock)
	elseif id == 3 then
		call TriggerAddAction(trig, function ActivateChainLightning)
	elseif id == 4 then
		call TriggerAddAction(trig, function ActivateRagingStrike)
	elseif id == 5 then
		call TriggerAddAction(trig, function ActivateWhirlwind)
	elseif id == 6 then
		call TriggerAddAction(trig, function ActivateIceShards)
	elseif id == 7 then
		call TriggerAddAction(trig, function ActivateDuel)
	elseif id == 8 then
		call TriggerAddAction(trig, function ActivateRavage)	
	elseif id == 9 then
		call TriggerAddAction(trig, function ActivateSummonFireElementals)
	elseif id == 10 then
		call TriggerAddAction(trig, function ActivateIndestructible)
	elseif id == 11 then
		call TriggerAddAction(trig, function ActivateChallenge)
	elseif id == 12 then
		call TriggerAddAction(trig, function ActivateWillOfTheWilds)
	endif
endfunction




////////////////////
// IMPROVE COMBAT ABILITIES at the cost of gold. Gets more expensive with each upgrade
function UpdateAbilityImproveTooltip takes integer id returns nothing
	local integer amount = 75 + abilityCurrentLevel[id] * 100

	if (id >= 10) then
		set amount = 100 + abilityCurrentLevel[id] * 200
	endif

	call SetCombatAbility(id, abilityCode[id], abilityUpgrade[id], abilityIconName[id], abilitySuperMaxLevel[id])

	//call DisplayTextToForce( GetPlayersAll(), I2S(levels) )

	if (abilityCurrentLevel[id] > 0 and abilityCurrentLevel[id] < abilitySuperMaxLevel[id]) then	
		call BlzFrameSetText(abilityImproveTooltipTitleFh[id], "Upgrade to |cffffcc00Level " + I2S(abilityCurrentLevel[id] + 1) + "|r")
		if (abilityImproveTooltipGoldIcon[id] == null) then
			set abilityImproveTooltipGoldIcon[id] = CreateTooltipIcon("goldIcon", abilityImproveTooltip[id],  abilityImproveTooltip[id], FRAMEPOINT_CENTER, -0.05, -0.004, "ToolTipGoldIcon.blp")
		endif

		if (abilityImproveTooltipGoldText[id] == null) then
			set abilityImproveTooltipGoldText[id] = CreateTooltipAmount("goldText", abilityImproveTooltip[id], abilityImproveTooltipGoldIcon[id], amount)
		else
			call BlzFrameSetText(abilityImproveTooltipGoldText[id], "|cffffcc00" + I2S(amount) + "|r")
		endif
	
	endif

	if (abilityCurrentLevel[id] > 0 and abilityCurrentLevel[id] < abilityMaxLevel[id]) then
		call BlzFrameSetTexture(abilityImproveFh[id], "ReplaceableTextures\\CommandButtons\\BTNSkillz.blp", 0, true)
	else
		call BlzFrameSetTexture(abilityImproveFh[id], "ReplaceableTextures\\CommandButtonsDisabled\\DISBTNSkillz.blp", 0, true)
	endif

	if (abilityCurrentLevel[id] == abilitySuperMaxLevel[id]) then
		call BlzDestroyFrame(abilityImproveButton[id])
	endif

endfunction

// Create tooltip for the next level of each ability
function CreateAbilityImproveTooltip takes integer id returns nothing
	call SetCombatAbility(id, abilityCode[id], abilityUpgrade[id], abilityIconName[id], abilitySuperMaxLevel[id])
	if (abilityCurrentLevel[id] > 0 and abilityCurrentLevel[id] < abilitySuperMaxLevel[id]) then
		set abilityImproveTooltipGoldIcon[id] = CreateTooltipIcon("goldIcon", abilityImproveTooltip[id],  abilityImproveTooltip[id], FRAMEPOINT_CENTER, -0.05, -0.004, "ToolTipGoldIcon.blp")
		if (id < 10) then
			set abilityImproveTooltipGoldText[id] = CreateTooltipAmount("goldText", abilityImproveTooltip[id], abilityImproveTooltipGoldIcon[id], 75 + abilityCurrentLevel[id] * 100)
		else
			set abilityImproveTooltipGoldText[id] = CreateTooltipAmount("goldText", abilityImproveTooltip[id], abilityImproveTooltipGoldIcon[id], 100 + abilityCurrentLevel[id] * 200)
		endif
    endif
    call UpdateAbilityImproveTooltip(id)
endfunction


// Unlocks an ability. Called at map start or when completing certain objectives
function UnlockAbility takes integer id returns nothing
	call SetPlayerTechMaxAllowedSwap( abilityUpgrade[id], 1, Player(0) )
    call SetPlayerTechResearchedSwap( abilityUpgrade[id], 1, Player(0) )
    set abilityCurrentLevel[id] = 1
    set abilityMaxLevel[id] = 1
    call UpdateAbilityTooltip(id)
    call UpdateAbilityImproveTooltip(id)
endfunction


// Improves an ability. Can be free under certain conditions (i.e. at map start)
function AbilityImprove takes integer id, integer levels, boolean isFree returns nothing
	local integer currentGold = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD)
	local integer requiredGold

	if (levels > 0) then
		if (isFree == false) then
			if (id < 10) then
				set requiredGold = 75 + abilityCurrentLevel[id] * 100 * levels
			else
				set requiredGold = 100 + abilityCurrentLevel[id] * 200 * levels
			endif
		else
			set requiredGold = 0
		endif

		if ((abilityCurrentLevel[id] > 0 and abilityMaxLevel[id] > 1 and abilityCurrentLevel[id] < abilityMaxLevel[id]) or isFree == true) then //otherwise don't react to player input
			if ( currentGold >= requiredGold) then
				call SetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD, currentGold - requiredGold)
				set abilityCurrentLevel[id] = abilityCurrentLevel[id] + levels
				call SetPlayerTechResearchedSwap( abilityUpgrade[id], abilityCurrentLevel[id], Player(0) )
				call UpdateAbilityTooltip(id)
				call UpdateAbilityImproveTooltip(id)

				if (isFree == false) then
					call PlaySoundBJ( gg_snd_GruntUpgradeComplete1 )
				endif
			else
		    	call PlaySoundBJ( gg_snd_GruntNoGold1 )
		    endif
	    endif
	endif
	
endfunction

//Functions that connect upgraded versions of an ability to the system
function AbilityImproveKnockback takes nothing returns nothing
    call AbilityImprove(1, 1, false)
endfunction

function AbilityImproveEarthshock takes nothing returns nothing
    call AbilityImprove(2, 1, false)
endfunction

function AbilityImproveChainLightning takes nothing returns nothing
    call AbilityImprove(3, 1, false)
endfunction

function AbilityImproveRagingStrike takes nothing returns nothing
    call AbilityImprove(4, 1, false)
endfunction

function AbilityImproveWhirlwind takes nothing returns nothing
    call AbilityImprove(5, 1, false)
endfunction

function AbilityImproveIceShards takes nothing returns nothing
    call AbilityImprove(6, 1, false)
endfunction

function AbilityImproveDuel takes nothing returns nothing
    call AbilityImprove(7, 1, false)
endfunction

function AbilityImproveRavage takes nothing returns nothing
    call AbilityImprove(8, 1, false)
endfunction

function AbilityImproveSummonFireElementals takes nothing returns nothing
    call AbilityImprove(9, 1, false)
endfunction

function AbilityImproveIndestructible takes nothing returns nothing
    call AbilityImprove(10, 1, false)
endfunction

function AbilityImproveChallenge takes nothing returns nothing
    call AbilityImprove(11, 1, false)
endfunction

function AbilityImproveWillOfTheWilds takes nothing returns nothing
    call AbilityImprove(12, 1, false)
endfunction


//Creates icon that a player presses to improve an ability and connects it to an event
function CreateAbilityPlus takes integer id returns nothing
	local trigger trig = CreateTrigger()

    //Button
    set abilityImproveButton[id] = BlzCreateFrameByType("GLUEBUTTON", "FaceButton", background, "IconButtonTemplate", 0)
    call BlzFrameSetAbsPoint(abilityImproveButton[id], FRAMEPOINT_CENTER, abilityPosX[id] + 0.035, abilityPosY[id])
    call BlzFrameSetSize(abilityImproveButton[id], 0.025, 0.025)

    //Icon (image set at update)
    set abilityImproveFh[id] = BlzCreateFrameByType("BACKDROP", "FaceButtonIcon", abilityImproveButton[id], "", 0)
    call BlzFrameSetAllPoints(abilityImproveFh[id], abilityImproveButton[id])

     //Tooltip
    set abilityImproveTooltip[id] = BlzCreateFrame("BoxedText", abilityImproveFh[id], 0, 0)
    call BlzFrameSetPoint(abilityImproveTooltip[id], FRAMEPOINT_BOTTOM, abilityImproveButton[id], FRAMEPOINT_TOP, 0.0, 0.0)
	call BlzFrameSetSize(abilityImproveTooltip[id], 0.11, 0.04)
	call BlzFrameSetTooltip(abilityImproveButton[id], abilityImproveTooltip[id])

	set abilityImproveTooltipTitleFh[id] = BlzCreateFrameByType("TEXT", "", abilityImproveTooltip[id], "BoxedTextTitle", 0)
	call BlzFrameSetPoint(abilityImproveTooltipTitleFh[id], FRAMEPOINT_LEFT, abilityImproveTooltip[id], FRAMEPOINT_LEFT, 0.005, 0.009)
	call BlzFrameSetText(abilityImproveTooltipTitleFh[id], "Upgrade |cffffcc00not available|r")

    //Action
    call BlzTriggerRegisterFrameEvent(trig, abilityImproveButton[id], FRAMEEVENT_CONTROL_CLICK)
    
    if id == 1 then
		call TriggerAddAction(trig, function AbilityImproveKnockback)
	elseif id == 2 then
		call TriggerAddAction(trig, function AbilityImproveEarthshock)
	elseif id == 3 then
		call TriggerAddAction(trig, function AbilityImproveChainLightning)
	elseif id == 4 then
		call TriggerAddAction(trig, function AbilityImproveRagingStrike)
	elseif id == 5 then
		call TriggerAddAction(trig, function AbilityImproveWhirlwind)
	elseif id == 6 then
		call TriggerAddAction(trig, function AbilityImproveIceShards)
	elseif id == 7 then
		call TriggerAddAction(trig, function AbilityImproveDuel)
	elseif id == 8 then
		call TriggerAddAction(trig, function AbilityImproveRavage)
	elseif id == 9 then
		call TriggerAddAction(trig, function AbilityImproveSummonFireElementals)
	elseif id == 10 then
		call TriggerAddAction(trig, function AbilityImproveIndestructible)
	elseif id == 11 then
		call TriggerAddAction(trig, function AbilityImproveChallenge)
	elseif id == 12 then
		call TriggerAddAction(trig, function AbilityImproveWillOfTheWilds)
	endif

endfunction

// Loop to create all buttons, checkboxes and tooltips
function CreateAllAbilityButtons takes nothing returns nothing
	local integer i = 1
	loop
	    exitwhen i > 12
	    call CreateAbilityButton(i)
	    call CreateAbilityCheckbox(i, abilityActive[i])
	    call CreateAbilityPlus(i)
	    call CreateAbilityImproveTooltip(i)
	    set i = i + 1
	endloop
	
endfunction


// Increases maximum level of an ability (i.e. when a player completes a certain objective)
function IncreaseAbilityMax takes integer id, integer max returns nothing
	call SetPlayerTechMaxAllowedSwap( abilityUpgrade[id], max, Player(0) )
	set abilityMaxLevel[id] = max
	call UpdateAbilityImproveTooltip(id)
endfunction


////////////////////
// RESEARCH UPGRADES
// Used by upgrades to to identify gold/lumber cost
function SetResearchUpgrade takes integer id, integer upgr, string btn, integer goldCost, integer lumberCost returns nothing
	set researchCode[id] = upgr
	set researchIconName[id] = btn
	set researchGoldCost[id] = goldCost
	set researchLumberCost[id] = lumberCost

	if (researchUnlocked[id] == true) then
		set researchIcon[id] = "ReplaceableTextures\\CommandButtons\\" + btn
	else
		set researchIcon[id] = "ReplaceableTextures\\CommandButtonsDisabled\\DIS" + btn
	endif
endfunction

//Decides which upgrades are part of the system and sets gold/lumber cost
function InitResearchUpgrades takes nothing returns nothing
	local real diff = 0.045
	//Berserker Strength
	call SetResearchUpgrade(1, 'R60N', "BTNBerserk.blp", 200, 200)
	set researchTitle[1] = "Research Berserker Strength"
	set researchText[1] = "Increases the hit points of Thrall by 100."
	set researchPosX[1] = gPosX - 0.145
	set researchPosY[1] = gPosY - 0.035

	//Pillage
	call SetResearchUpgrade(2, 'R60K', "BTNPillage.blp", 300, 300)
	set researchTitle[2] = "Research Pillage"
	set researchText[2] = "Causes Thrall's attacks to gain resources when hitting enemy buildings."
	set researchPosX[2] = researchPosX[1]
	set researchPosY[2] = researchPosY[1] - diff

	//Resistant Skin
	call SetResearchUpgrade(3, 'R60Z', "BTNResistantSkin.blp", 400, 400)
	set researchTitle[3] = "Research Resistant Skin"
	set researchText[3] = "Gives Thrall increased resistance to spell damage."
	set researchPosX[3] = researchPosX[2]
	set researchPosY[3] = researchPosY[2] - diff

	//Mining
	call SetResearchUpgrade(4, 'R60Y', "BTNGoldMine.blp", 300, 300)
	set researchTitle[4] = "Research Mining"
	set researchText[4] = "Allows Thrall to create gold when destroying rock chunks."
	set researchPosX[4] = researchPosX[3]
	set researchPosY[4] = researchPosY[3] - diff

	//Ultravision
	call SetResearchUpgrade(5, 'R60L', "BTNUltravision.blp", 275, 275)
	set researchTitle[5] = "Research Ultravision"
	set researchText[5] = "Gives Thrall the ability to see as far at night as he does during the day."
	set researchPosX[5] = researchPosX[1] + diff
	set researchPosY[5] = researchPosY[1]

	//Cut Down Trees
	call SetResearchUpgrade(6, 'R60J', "BTNGatherGold.blp", 400, 400)
	set researchTitle[6] = "Research Cut Down Trees"
	set researchText[6] = "Allows Thrall to harvest lumber by destroying trees."
	set researchPosX[6] = researchPosX[5]
	set researchPosY[6] = researchPosY[5] - diff

	//Companion Backpack
	call SetResearchUpgrade(7, 'R60M', "BTNPackBeast.blp", 200, 200)
	set researchTitle[7] = "Research Companion Backpack"
	set researchText[7] = "Gives Thrall's companion the ability to carry items."
	set researchPosX[7] = researchPosX[6]
	set researchPosY[7] = researchPosY[6] - diff

	//Gift of Fire
	call SetResearchUpgrade(8, 'R60Q', "BTNFire.blp", 500, 500)
	set researchTitle[8] = "Research Gift of Fire"
	set researchText[8] = "Increases the attack damage of Thrall by 8."
	set researchPosX[8] = researchPosX[7]
	set researchPosY[8] = researchPosY[7] - diff

	//Gift of Air
	call SetResearchUpgrade(9, 'R60W', "BTNTornado.blp", 500, 500)
	set researchTitle[9] = "Research Gift of Air"
	set researchText[9] = "Increases the movement speed of Thrall."
	set researchPosX[9] = researchPosX[8] + diff
	set researchPosY[9] = researchPosY[1]

	//Gift of Earth
	call SetResearchUpgrade(10, 'R60V', "BTNEarthquake.blp", 500, 500)
	set researchTitle[10] = "Research Gift of Earth"
	set researchText[10] = "Thrall has a 10 percent chance to to slow an enemy unit when he attacks.|nLasts 5 seconds."
	set researchPosX[10] = researchPosX[9]
	set researchPosY[10] = researchPosY[9] - diff

	//Gift of Water
	call SetResearchUpgrade(11, 'R60U', "BTNSummonWaterElemental.blp", 500, 500)
	set researchTitle[11] = "Research Gift of Water"
	set researchText[11] = "Increases the mana capacity of Thrall by 100."
	set researchPosX[11] = researchPosX[10]
	set researchPosY[11] = researchPosY[10] - diff

	//Gift of the Wilds
	call SetResearchUpgrade(12, 'R60X', "BTNPolymorph.blp", 500, 500)
	set researchTitle[12] = "Research Gift of the Wilds"
	set researchText[12] = "Enemy units have a 10 percent chance to be morphed into critters when they attack.|nLasts 3 seconds."
	set researchPosX[12] = researchPosX[11]
	set researchPosY[12] = researchPosY[11] - diff

	//Food Bonus +2
	call SetResearchUpgrade(13, 'R60P', "BTNHex.blp", 200, 200)
	set researchTitle[13] = "Research Food Bonus +2"
	set researchText[13] = "Increases Food Cap by 2."
	set researchPosX[13] = researchPosX[11] + diff
	set researchPosY[13] = researchPosY[1]

	//Food Bonus +3
	call SetResearchUpgrade(14, 'R613', "BTNSheep.blp", 300, 300)
	set researchTitle[14] = "Research Food Bonus +3"
	set researchText[14] = "Increases Food Cap by 3."
	set researchPosX[14] = researchPosX[13]
	set researchPosY[14] = researchPosY[13] - diff

	//Food Bonus +4
	call SetResearchUpgrade(15, 'R614', "BTNStag.blp", 400, 400)
	set researchTitle[15] = "Research Food Bonus +4"
	set researchText[15] = "Increases Food Cap by 4."
	set researchPosX[15] = researchPosX[14]
	set researchPosY[15] = researchPosY[14] - diff

endfunction


//Create tooltip for upgrades
function CreateResearchTooltip takes integer id returns nothing
	local framehandle goldIcon
	local framehandle goldText
	local framehandle lumberIcon
	local framehandle lumberText

	set researchTooltip[id] = BlzCreateFrame("BoxedText", researchBackdrop[id], 0, 0)
	
	call BlzFrameSetTooltip(researchHover[id], researchTooltip[id]) 
	call BlzFrameSetPoint(researchTooltip[id], FRAMEPOINT_BOTTOM, researchBackdrop[id], FRAMEPOINT_TOP, 0.0, 0.0)
	call BlzFrameSetSize(researchTooltip[id], 0.18, 0.09)
	call BlzFrameSetText(BlzGetFrameByName("BoxedTextTitle", 0), researchTitle[id])
	call BlzFrameSetText(BlzGetFrameByName("BoxedTextValue", 0), "|n" + researchText[id])

	set goldIcon = CreateTooltipIcon("goldIcon", researchTooltip[id],  researchTooltip[id], FRAMEPOINT_CENTER, -0.085, 0.022, "ToolTipGoldIcon.blp")
	set goldText = CreateTooltipAmount("goldText", researchTooltip[id], goldIcon, researchGoldCost[id])

	set lumberIcon = CreateTooltipIcon("lumberIcon", researchTooltip[id],  goldText, FRAMEPOINT_RIGHT, 0.007, 0.0, "ToolTipLumberIcon.blp")
	set lumberText = CreateTooltipAmount("lumberText", researchTooltip[id], lumberIcon, researchLumberCost[id])

endfunction

// Create checkbox for upgrades
function CreateResearchCheckbox takes integer id returns nothing
	set researchCheckbox[id] = BlzCreateFrameByType("BACKDROP", "Face", background, "", 0)
	call BlzFrameSetSize(researchCheckbox[id], 0.02, 0.02)
	call BlzFrameSetAbsPoint(researchCheckbox[id], FRAMEPOINT_CENTER, researchPosX[id] + 0.015, researchPosY[id] -0.015)
	call BlzFrameSetTexture(researchCheckbox[id], "UI\\Widgets\\EscMenu\\Human\\checkbox-check.blp", 0, true)
endfunction

// Special upgrade that increases current food supply for the player
function ResearchUpgradeFood takes nothing returns integer
	local integer baseFood = 5
	if (researchApplied[13] == true) then
		set baseFood = baseFood + 2
	endif
	if (researchApplied[14] == true) then
		set baseFood = baseFood + 3
	endif
	if (researchApplied[15] == true) then
		set baseFood = baseFood + 4
	endif
	return baseFood
endfunction


// Research an upgrade at the cost of money
function ResearchUpgrade takes integer id, boolean isFree returns nothing
	local integer currentGold = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD)
	local integer currentLumber = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_LUMBER)
	local integer requiredGold = researchGoldCost[id]
	local integer requiredLumber = researchLumberCost[id]

	if (isFree == true) then
		set requiredGold = 0
		set requiredLumber = 0
	endif

	if ( (researchUnlocked[id] == true and researchApplied[id] == false) or isFree == true) then //otherwise don't react to player input
		if ( currentGold >= researchGoldCost[id] or isFree == true) then
			if ( currentLumber >= researchLumberCost[id] or isFree == true) then

				call SetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD, currentGold - requiredGold)
				call SetPlayerState(Player(0), PLAYER_STATE_RESOURCE_LUMBER, currentLumber - requiredLumber)

				call SetPlayerTechResearchedSwap( researchCode[id], 1, Player(0) )
				set researchApplied[id] = true
				call CreateResearchCheckbox(id)
				if (id == 13 or id == 14 or id == 15) then //Food Bonus must be applied manually
					call SetPlayerStateBJ( Player(0), PLAYER_STATE_RESOURCE_FOOD_CAP, ResearchUpgradeFood() )
				endif
				if (isFree == false) then
					call PlaySoundBJ( gg_snd_GruntResearchComplete1 )
				endif
			else
				call PlaySoundBJ( gg_snd_GruntNoLumber1 )
			endif
		else
	    	call PlaySoundBJ( gg_snd_GruntNoGold1 )
	    endif
    endif

endfunction

// Connect certain upgrades to the system
function ResearchBerserkerStrength takes nothing returns nothing
    call ResearchUpgrade(1, false)
endfunction

function ResearchPillage takes nothing returns nothing
    call ResearchUpgrade(2, false)
endfunction

function ResearchResistantSkin takes nothing returns nothing
    call ResearchUpgrade(3, false)
endfunction

function ResearchMining takes nothing returns nothing
    call ResearchUpgrade(4, false)
endfunction

function ResearchUltravision takes nothing returns nothing
    call ResearchUpgrade(5, false)
endfunction

function ResearchCutDownTrees takes nothing returns nothing
    call ResearchUpgrade(6, false)
endfunction

function ResearchCompanionBackpack takes nothing returns nothing
    call ResearchUpgrade(7, false)
endfunction

function ResearchGiftOfFire takes nothing returns nothing
    call ResearchUpgrade(8, false)
endfunction

function ResearchGiftOfAir takes nothing returns nothing
    call ResearchUpgrade(9, false)
endfunction

function ResearchGiftOfEarth takes nothing returns nothing
    call ResearchUpgrade(10, false)
endfunction

function ResearchGiftOfWater takes nothing returns nothing
    call ResearchUpgrade(11, false)
endfunction

function ResearchGiftOfTheWilds takes nothing returns nothing
    call ResearchUpgrade(12, false)
endfunction

function ResearchFoodBonus2 takes nothing returns nothing
    call ResearchUpgrade(13, false)
endfunction

function ResearchFoodBonus3 takes nothing returns nothing
    call ResearchUpgrade(14, false)
endfunction

function ResearchFoodBonus4 takes nothing returns nothing
    call ResearchUpgrade(15, false)
endfunction


// Create research button and add events
function CreateResearchButton takes integer id returns nothing
	local trigger trig = CreateTrigger()

	call SetResearchUpgrade(id, researchCode[id], researchIconName[id], researchGoldCost[id], researchLumberCost[id])

	//Icon
	set researchBackdrop[id] = BlzCreateFrameByType("BACKDROP", "Face", background, "", 0)
	call BlzFrameSetSize(researchBackdrop[id], 0.04, 0.04)
	call BlzFrameSetTexture(researchBackdrop[id], researchIcon[id], 0, true)
	call BlzFrameSetAbsPoint(researchBackdrop[id], FRAMEPOINT_CENTER, researchPosX[id], researchPosY[id])

	//Button
	set researchHover[id] = BlzCreateFrameByType("GLUEBUTTON", "FaceFrame", researchBackdrop[id],"IconButtonTemplate", 0)
	call BlzFrameSetAllPoints(researchHover[id], researchBackdrop[id]) 
	
	//Tooltip
	call CreateResearchTooltip(id)


	//Action
	call BlzTriggerRegisterFrameEvent(trig, researchHover[id], FRAMEEVENT_CONTROL_CLICK)
	
	if id == 1 then
		call TriggerAddAction(trig, function ResearchBerserkerStrength)
	elseif id == 2 then
		call TriggerAddAction(trig, function ResearchPillage)
	elseif id == 3 then
		call TriggerAddAction(trig, function ResearchResistantSkin)
	elseif id == 4 then
		call TriggerAddAction(trig, function ResearchMining)
	elseif id == 5 then
		call TriggerAddAction(trig, function ResearchUltravision)
	elseif id == 6 then
		call TriggerAddAction(trig, function ResearchCutDownTrees)
	elseif id == 7 then
		call TriggerAddAction(trig, function ResearchCompanionBackpack)
	elseif id == 8 then
		call TriggerAddAction(trig, function ResearchGiftOfFire)
	elseif id == 9 then
		call TriggerAddAction(trig, function ResearchGiftOfAir)
	elseif id == 10 then
		call TriggerAddAction(trig, function ResearchGiftOfEarth)
	elseif id == 11 then
		call TriggerAddAction(trig, function ResearchGiftOfWater)
	elseif id == 12 then
		call TriggerAddAction(trig, function ResearchGiftOfTheWilds)
	elseif id == 13 then
		call TriggerAddAction(trig, function ResearchFoodBonus2)
	elseif id == 14 then
		call TriggerAddAction(trig, function ResearchFoodBonus3)
	elseif id == 15 then
		call TriggerAddAction(trig, function ResearchFoodBonus4)
	endif
endfunction



// Unlocks a research (i.e. after completing a certain quest)
function UnlockResearch takes integer id returns nothing
	call SetPlayerTechMaxAllowedSwap( researchCode[id], 1, Player(0) )
	set researchUnlocked[id] = true
	call SetResearchUpgrade(id, researchCode[id], researchIconName[id], researchGoldCost[id], researchLumberCost[id])
	call BlzFrameSetTexture(researchBackdrop[id], researchIcon[id], 0, true)
endfunction

// Create all research buttons
function CreateAllResearchButtons takes nothing returns nothing
	local integer i = 1
	loop
	    exitwhen i > 15
	    call CreateResearchButton(i)
	    set i = i + 1
	endloop
	
endfunction


////////////////////
// SUMMON COMPANION
//Used to decide which units are part of the companion system
function SetCompanion takes integer id, integer comp, string btn, integer goldCost, integer lumberCost, integer foodCost returns nothing
	set companionCode[id] = comp
	set companionIconName[id] = btn
	set companionGoldCost[id] = goldCost
	set companionLumberCost[id] = lumberCost
	set companionFoodCost[id] = foodCost

	if (companionUnlocked[id] == true) then
		set companionIcon[id] = "ReplaceableTextures\\CommandButtons\\" + btn
	else
		set companionIcon[id] = "ReplaceableTextures\\CommandButtonsDisabled\\DIS" + btn
	endif
endfunction

//Decides which units are part of the companion system. Also adds gold/lumber/food cost
function InitCompanions takes nothing returns nothing
	local real diff = 0.045
	//Dog
	call SetCompanion(1, 'n601', "BTNWolf.blp", 25, 50, 1)
	set companionTitle[1] = "Summon Dog"
	set companionText[1] = "Fast scout that can decrease enemy unit damage for a short duration. Only one can exist at a time."
	set companionPosX[1] = gPosX + 0.055
	set companionPosY[1] = gPosY - 0.035

	//Dark Wolf
	call SetCompanion(2, 'n61Y', "BTNDireWolf.blp", 25, 100, 2)
	set companionTitle[2] = "Summon Dark Wolf"
	set companionText[2] = "Light melee unit that can increase damage of nearby allied units. Only one can exist at a time."
	set companionPosX[2] = companionPosX[1]
	set companionPosY[2] = companionPosY[1] - diff

	//Snowsong
	call SetCompanion(3, 'n61Z', "BTNTimberWolf.blp", 100, 250, 5)
	set companionTitle[3] = "Summon Snowsong"
	set companionText[3] = "Heavy melee unit that can stun nearby units. Comes back to life when killed. Only one can exist at a time."
	set companionPosX[3] = companionPosX[1]
	set companionPosY[3] = companionPosY[2] - diff

	//Grunt
	call SetCompanion(4, 'o61F', "BTNGrunt.blp", 50, 200, 3)
	set companionTitle[4] = "Summon Grunt"
	set companionText[4] = "Brutish Orc warrior."
	set companionPosX[4] = companionPosX[1] + diff
	set companionPosY[4] = companionPosY[1]

	//Spearthrower
	call SetCompanion(5, 'n620', "BTNChaosWarlockGreen.blp", 30, 200, 2)
	set companionTitle[5] = "Summon Spearthrower"
	set companionText[5] = "Medium ranged unit that throws envenomed spears."
	set companionPosX[5] = companionPosX[1] + diff
	set companionPosY[5] = companionPosY[2]

	//Raider
	call SetCompanion(6, 'o61I', "BTNRaider.blp", 40, 180, 3)
	set companionTitle[6] = "Summon Raider"
	set companionText[6] = "Highly mobile wolf rider. Effective against buildings. Has the Ensnare ability."
	set companionPosX[6] = companionPosX[1] + diff
	set companionPosY[6] = companionPosY[3]

	//Catapult
	call SetCompanion(7, 'o000', "BTNCatapult.blp", 100, 250, 4)
	set companionTitle[7] = "Summon Catapult"
	set companionText[7] = "Long-range siege weaponry. Effective against buildings but slow and vulnerable. Can self-repair."
	set companionPosX[7] = companionPosX[4] + diff
	set companionPosY[7] = companionPosY[1]

	//Shaman
	call SetCompanion(8, 'o61R', "BTNShaman.blp", 20, 150, 2)
	set companionTitle[8] = "Summon Shaman"
	set companionText[8] = "Primary spellcaster. Can initially cast Purge, which slows and dispels positive and negative buffs. Also has the Lightning Shield and Bloodlust abilities."
	set companionPosX[8] = companionPosX[4] + diff
	set companionPosY[8] = companionPosY[2]

	//Goblin Shredder
	call SetCompanion(9, 'n62R', "BTNJunkGolem.blp", 100, 300, 4)
	set companionTitle[9] = "Summon Goblin Shredder"
	set companionText[9] = "Medium melee unit made of iron. Can destroy trees and self-repair."
	set companionPosX[9] = companionPosX[4] + diff
	set companionPosY[9] = companionPosY[3]

endfunction

//Create tooltip for each companion
function CreateCompanionTooltip takes integer id returns nothing
	local framehandle goldIcon
	local framehandle goldText
	local framehandle lumberIcon
	local framehandle lumberText
	local framehandle foodIcon
	local framehandle foodText

	set companionTooltip[id] = BlzCreateFrame("BoxedText", companionBackdrop[id], 0, 0)
	
	call BlzFrameSetTooltip(companionHover[id], companionTooltip[id]) 
	call BlzFrameSetPoint(companionTooltip[id], FRAMEPOINT_BOTTOM, companionBackdrop[id], FRAMEPOINT_TOP, 0.0, 0.0)
	call BlzFrameSetSize(companionTooltip[id], 0.18, 0.09)
	call BlzFrameSetText(BlzGetFrameByName("BoxedTextTitle", 0), companionTitle[id])
	call BlzFrameSetText(BlzGetFrameByName("BoxedTextValue", 0), "|n" + companionText[id])

	set goldIcon = CreateTooltipIcon("goldIcon", companionTooltip[id],  companionTooltip[id], FRAMEPOINT_CENTER, -0.085, 0.022, "ToolTipGoldIcon.blp")
	set goldText = CreateTooltipAmount("goldText", companionTooltip[id], goldIcon, companionGoldCost[id])

	if (companionLumberCost[id] > 0) then
		set lumberIcon = CreateTooltipIcon("lumberIcon", companionTooltip[id],  goldText, FRAMEPOINT_RIGHT, 0.007, 0.0, "ToolTipLumberIcon.blp")
		set lumberText = CreateTooltipAmount("lumberText", companionTooltip[id], lumberIcon, companionLumberCost[id])
		set foodIcon = CreateTooltipIcon("foodIcon", companionTooltip[id],  lumberText, FRAMEPOINT_RIGHT, 0.007, 0.0, "ToolTipSupplyIcon.blp")
	else
		set foodIcon = CreateTooltipIcon("foodIcon", companionTooltip[id],  goldText, FRAMEPOINT_RIGHT, 0.007, 0.0, "ToolTipSupplyIcon.blp")
	endif

	set foodText = CreateTooltipAmount("foodText", companionTooltip[id], foodIcon, companionFoodCost[id])

endfunction

// Special companion that can only exist once per type
function CanSummonSnowsong takes nothing returns boolean
	return (CountLivingPlayerUnitsOfTypeId('n61Z', Player(0)) == 0)
endfunction

// Special companion that can only exist once per type
function CanSummonDarkWolf takes nothing returns boolean
	return (CountLivingPlayerUnitsOfTypeId('n61Y', Player(0)) == 0)
endfunction

// Special companion that can only exist once per type
function CanSummonDog takes nothing returns boolean
	return (CountLivingPlayerUnitsOfTypeId('n601', Player(0)) == 0)
endfunction


// Called when summoning a companion
function SummonCompanion takes integer id returns nothing
	local integer currentGold = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD)
	local integer currentLumber = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_LUMBER)
	local integer currentFoodCap = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_FOOD_CAP)
	local integer currentFoodUsed = GetPlayerState(Player(0), PLAYER_STATE_RESOURCE_FOOD_USED)

	if (companionUnlocked[id] == true) then //otherwise don't react to player input
		
		if ((id == 1 and CanSummonDog() == false) or (id == 2 and CanSummonDarkWolf() == false) or (id == 3 and CanSummonSnowsong() == false)) then //Dog/Dark Wolf/Snowsong can exist only once
			call PlaySoundBJ( gg_snd_Error )
		elseif ( (companionFoodCost[id] + currentFoodUsed) <= currentFoodCap ) then
			if ( currentGold >= companionGoldCost[id]) then
				if ( currentLumber >= companionLumberCost[id]) then

					call SetPlayerState(Player(0), PLAYER_STATE_RESOURCE_GOLD, currentGold - companionGoldCost[id])
					call SetPlayerState(Player(0), PLAYER_STATE_RESOURCE_LUMBER, currentLumber - companionLumberCost[id])
					call CreateNUnitsAtLoc( 1, companionCode[id], Player(0), GetUnitLoc(udg_Thrall), bj_UNIT_FACING )
					call AddSpecialEffectTargetUnitBJ( "origin", GetLastCreatedUnit(), "Abilities\\Spells\\Orc\\FeralSpirit\\feralspirittarget.mdl" )
				else
					call PlaySoundBJ( gg_snd_GruntNoLumber1 )
				endif
			else
		    	call PlaySoundBJ( gg_snd_GruntNoGold1 )
		    endif
		else
			call PlaySoundBJ( gg_snd_GruntNoFood1 )
		endif

    endif

endfunction

//Add various unit types to the companion system
function SummonDog takes nothing returns nothing
    call SummonCompanion(1)
endfunction

function SummonDireWolf takes nothing returns nothing
    call SummonCompanion(2)
endfunction

function SummonSnowsong takes nothing returns nothing
    call SummonCompanion(3)
endfunction

function SummonGrunt takes nothing returns nothing
    call SummonCompanion(4)
endfunction

function SummonSpearthrower takes nothing returns nothing
    call SummonCompanion(5)
endfunction

function SummonRaider takes nothing returns nothing
    call SummonCompanion(6)
endfunction

function SummonCatapult takes nothing returns nothing
    call SummonCompanion(7)
endfunction

function SummonShaman takes nothing returns nothing
    call SummonCompanion(8)
endfunction

function SummonGoblinShredder takes nothing returns nothing
    call SummonCompanion(9)
endfunction

// Create button for each companion
function CreateCompanionButton takes integer id returns nothing
	local trigger trig = CreateTrigger()

	call SetCompanion(id, companionCode[id], companionIconName[id], companionGoldCost[id], companionLumberCost[id], companionFoodCost[id])

	//Icon
	set companionBackdrop[id] = BlzCreateFrameByType("BACKDROP", "Face", background, "", 0)
	call BlzFrameSetAbsPoint(companionBackdrop[id], FRAMEPOINT_CENTER, companionPosX[id], companionPosY[id])
	call BlzFrameSetSize(companionBackdrop[id], 0.04, 0.04)
	call BlzFrameSetTexture(companionBackdrop[id], companionIcon[id], 0, true)

	//Button
	set companionHover[id] = BlzCreateFrameByType("GLUEBUTTON", "FaceFrame", companionBackdrop[id],"IconButtonTemplate", 0)
	call BlzFrameSetAllPoints(companionHover[id], companionBackdrop[id]) 
	
	//Tooltip
	call CreateCompanionTooltip(id)

	//Action
	call BlzTriggerRegisterFrameEvent(trig, companionHover[id], FRAMEEVENT_CONTROL_CLICK)
	
	if id == 1 then
		call TriggerAddAction(trig, function SummonDog)
	elseif id == 2 then
		call TriggerAddAction(trig, function SummonDireWolf)
	elseif id == 3 then
		call TriggerAddAction(trig, function SummonSnowsong)
	elseif id == 4 then
		call TriggerAddAction(trig, function SummonGrunt)
	elseif id == 5 then
		call TriggerAddAction(trig, function SummonSpearthrower)
	elseif id == 6 then
		call TriggerAddAction(trig, function SummonRaider)
	elseif id == 7 then
		call TriggerAddAction(trig, function SummonCatapult)
	elseif id == 8 then
		call TriggerAddAction(trig, function SummonShaman)
	elseif id == 9 then
		call TriggerAddAction(trig, function SummonGoblinShredder)
	endif
endfunction

// Unlock companion (i.e. after completing a certain objective)
function UnlockCompanion takes integer id returns nothing
	set companionUnlocked[id] = true
	call SetCompanion(id, companionCode[id], companionIconName[id], companionGoldCost[id], companionLumberCost[id], companionFoodCost[id])
	call BlzFrameSetTexture(companionBackdrop[id], companionIcon[id], 0, true)
endfunction

// Loop to create all companion buttons
function CreateAllCompanionButtons takes nothing returns nothing
	local integer i = 1
	loop
	    exitwhen i > 9
	    call CreateCompanionButton(i)
	    set i = i + 1
	endloop
endfunction

//INITIALIZATION
// Create all basic elements
function InitMenu takes nothing returns nothing
	call loadTOC()
	call CreateBackground()
	call InitCombatAbilities()
	call CreateAllAbilityButtons()
	call InitResearchUpgrades()
	call CreateAllResearchButtons()
	call InitCompanions()
	call CreateAllCompanionButtons()
	call CreateCloseButton()
	call OpenOrCloseMenu(false)
endfunction


////////////////////////////
// Stores ability levels. Used for map-to-map-loading
function StoreAbilityLevels takes nothing returns nothing
local integer i = 1
loop
    exitwhen i > 12
		call StoreIntegerBJ( GetPlayerTechCountSimple(abilityUpgrade[i], Player(0)), "AbilityLevel" + I2S(i), "Category", GetLastCreatedGameCacheBJ() )
    set i = i + 1
endloop
endfunction

// Loads ability levels. Used for map-to-map-loading
function LoadAbilityLevels takes nothing returns nothing
	local integer i = 1
	local integer abCurLevel
	loop
	    exitwhen i > 12
	    	set abCurLevel = GetStoredIntegerBJ("AbilityLevel" + I2S(i), "Category", GetLastCreatedGameCacheBJ())
	    	if (abCurLevel > 1) then
	    		call AbilityImprove(i,  abCurLevel -1, true)
	    	endif
	    set i = i + 1
	endloop
endfunction


////////////////////////////
// Stores unlocked upgrades. Used for map-to-map-loading
function StoreUpgradeUnlocked takes nothing returns nothing
local integer i = 1
loop
    exitwhen i > 15
    	call StoreBooleanBJ( researchUnlocked[i], "UpgradeLevelMax" + I2S(i), "Category", GetLastCreatedGameCacheBJ() )
		call StoreBooleanBJ( researchApplied[i],  "UpgradeLevelCur" + I2S(i), "Category", GetLastCreatedGameCacheBJ() )
    set i = i + 1
endloop
endfunction

// Loads unlocked upgrades. Used for map-to-map-loading
function LoadUpgradeUnlocked takes nothing returns nothing
local integer i = 1
local boolean upgradeUnlocked
local boolean upgradeLevelCur
loop
    exitwhen i > 15
    	set upgradeUnlocked = GetStoredBooleanBJ("UpgradeLevelMax" + I2S(i), "Category", GetLastCreatedGameCacheBJ())
		set upgradeLevelCur = GetStoredBooleanBJ("UpgradeLevelCur" + I2S(i), "Category", GetLastCreatedGameCacheBJ())

		if (upgradeUnlocked == true) then
    		call UnlockResearch(i)
    	endif

    	if (upgradeLevelCur == true) then
    		if (i == 1 or i == 11) then //Don't re-apply: Berserker Strength, Gift of Water
    			call CreateResearchCheckbox(i)
    			set researchApplied[i] = true
    		else
    			call ResearchUpgrade(i, true)
    		endif
    	endif
    set i = i + 1
endloop
endfunction

// Loads researched upgrades. Used for map-to-map-loading
function LoadGameUpgradesResearched takes nothing returns nothing
	local integer i = 1
	loop
	    exitwhen i > 15
	    	if (researchApplied[i] == true) then
				call CreateResearchCheckbox(i)
	    	endif
	    set i = i + 1
	endloop
endfunction



////////////////////////////
// Stores unlocked companions. Used for map-to-map-loading
function StoreCompanionUnlocked takes nothing returns nothing
local integer i = 1
loop
    exitwhen i > 9
    	call StoreBooleanBJ( companionUnlocked[i], "CompanionUnlocked" + I2S(i), "Category", GetLastCreatedGameCacheBJ() )
    set i = i + 1
endloop
endfunction

// Loads unlocked companions. Used for map-to-map-loading
function LoadCompanionUnlocked takes nothing returns nothing
local integer i = 1
local boolean compUnlocked
loop
    exitwhen i > 9
    	set compUnlocked = GetStoredBooleanBJ("CompanionUnlocked" + I2S(i), "Category", GetLastCreatedGameCacheBJ())

		if (compUnlocked == true) then
    		call UnlockCompanion(i)
    	endif
    set i = i + 1
endloop
endfunction

////////////////////////////
// Loads active abilites.  Used for map-to-map-loading
function LoadActiveAbilities takes nothing returns nothing
	local string abilityCurrQ = GetStoredStringBJ("AbilityCurrentQ", "Category", GetLastCreatedGameCacheBJ())
	local string abilityCurrW = GetStoredStringBJ("AbilityCurrentW", "Category", GetLastCreatedGameCacheBJ())
	local string abilityCurrE = GetStoredStringBJ("AbilityCurrentE", "Category", GetLastCreatedGameCacheBJ())
	local string abilityCurrR = GetStoredStringBJ("AbilityCurrentR", "Category", GetLastCreatedGameCacheBJ())

    if ( abilityCurrQ == "knockback" ) then
        call ActivateKnockback()
    elseif ( abilityCurrQ == "earthshock" ) then
        call ActivateEarthshock()
    elseif ( abilityCurrQ == "chainlightning" ) then
        call ActivateChainLightning()
    endif

    if ( abilityCurrW == "ragingstrike" ) then
        call ActivateRagingStrike()
    elseif ( abilityCurrW == "whirlwind" ) then
        call ActivateWhirlwind()
    elseif ( abilityCurrW == "iceShards" ) then
        call ActivateIceShards()
    endif

    if ( abilityCurrE == "duel" ) then
        call ActivateDuel()
    elseif ( abilityCurrE == "ravage" ) then
        call ActivateRavage()
    elseif ( abilityCurrE == "summonfirelementals" ) then
        call ActivateSummonFireElementals()
    endif

    if ( abilityCurrR == "indestructible" ) then
        call ActivateIndestructible()
    elseif ( abilityCurrR == "challenge" ) then
        call ActivateChallenge()
    elseif ( abilityCurrR == "willofthewilds" ) then
        call ActivateWillOfTheWilds()
    endif
endfunction
