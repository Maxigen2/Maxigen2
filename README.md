fastPixelGetColorBufferDC = 0
fastPixelGetColorReady = 0
fastPixelGetColorWait = 1

fastPixelGetColor(x, y) {
	global fastPixelGetColorBufferDC, fastPixelGetColorReady, fastPixelGetColorWait
	global fastPixelGetColorScreenLeft, fastPixelGetColorScreenTop
	; check if there is a valid data buffer
	if (!fastPixelGetColorReady) {
		if (fastPixelGetColorWait) {
			Start := A_TickCount 
			While !fastPixelGetColorReady {
				Sleep, 10
				if (A_TickCount - Start > 5000)
					return -3	; time out if data is not ready after 5 seconds
			}
		}
		else
			return -2	; return an invalid color if waiting is disabled
	}
	pixel := DllCall("GetPixel", "Uint", fastPixelGetColorBufferDC, "int", x - fastPixelGetColorScreenLeft, "int", y - fastPixelGetColorScreenTop)
	return pixel
}

updateFastPixelGetColor() {
	global fastPixelGetColorReady, fastPixelGetColorBufferDC
	static oldObject = 0, hBuffer = 0
	static screenWOld = 0, screenHOld = 0
	; get screen dimensions
	global fastPixelGetColorScreenLeft, fastPixelGetColorScreenTop
	SysGet, fastPixelGetColorScreenLeft, 76
	SysGet, fastPixelGetColorScreenTop, 77
	SysGet, screenW, 78
	SysGet, screenH, 79
	fastPixelGetColorReady = 0
	; determine whether the old buffer can be reused
	bufferInvalid := screenW <> screenWOld OR screenH <> screenHOld OR fastPixelGetColorBufferDC = 0 OR hBuffer = 0
	screenWOld := screenW
	screenHOld := screenH
	if (bufferInvalid) {
		; cleanly discard the old buffer
		DllCall("SelectObject", "Uint", fastPixelGetColorBufferDC, "Uint", oldObject)
		DllCall("DeleteDC", "Uint", fastPixelGetColorBufferDC)
		DllCall("DeleteObject", "Uint", hBuffer)
		; create a new empty buffer
		fastPixelGetColorBufferDC := DllCall("CreateCompatibleDC", "Uint", 0)
		hBuffer := CreateDIBSection(fastPixelGetColorBufferDC, screenW, screenH)
		oldObject := DllCall("SelectObject", "Uint", fastPixelGetColorBufferDC, "Uint", hBuffer)
	}
	screenDC := DllCall("GetDC", "Uint", 0)
	; retrieve the whole screen into the newly created buffer
	DllCall("BitBlt", "Uint", fastPixelGetColorBufferDC, "int", 0, "int", 0, "int", screenW, "int", screenH, "Uint", screenDC, "int", fastPixelGetColorScreenLeft, "int", fastPixelGetColorScreenTop, "Uint", 0x40000000 | 0x00CC0020)
	; important: release the DC of the screen
	DllCall("ReleaseDC", "Uint", 0, "Uint", screenDC)
	fastPixelGetColorReady = 1
}

CreateDIBSection(hDC, nW, nH, bpp = 32, ByRef pBits = "") {
	NumPut(VarSetCapacity(bi, 40, 0), bi)
	NumPut(nW, bi, 4)
	NumPut(nH, bi, 8)
	NumPut(bpp, NumPut(1, bi, 12, "UShort"), 0, "Ushort")
	NumPut(0,  bi,16)
	Return DllCall("gdi32\CreateDIBSection", "Uint", hDC, "Uint", &bi, "Uint", 0, "UintP", pBits, "Uint", 0, "Uint", 0)
}
#NoEnv  ; Recommended for performance and compatibility with future AutoHotkey releases.
; #Warn  ; Enable warnings to assist with detecting common errors.
SendMode Input  ; Recommended for new scripts due to its superior speed and reliability.
SetWorkingDir %A_ScriptDir%  ; Ensures a consistent starting directory.
#SingleInstance force

#include fastPixelGetColor.ahk



;variables
putsomedelay := 1
StoreByColor := 1
SkipWhite := 0
SkipBlack := 0
SkipColored := 0
StoredImage := {}
CheckedImage := {}
ShowMousePosition := 0
DragTheMouse := 0
DoubleClickPaste := 1
CtrlAPaste := 1
RunningPasting := 0
RunningCoping := 0
RunningChecking := 0
ClickToSave := "FirstCopy"
ClickBeforeColoring := 0
SetDefaultMouseSpeed, 0
SavedMouseClickFirstCopyx := "Not Set"
SavedMouseClickFirstCopyy := "Not Set"
SavedMouseClickLastCopyx := "Not Set"
SavedMouseClickLastCopyy := "Not Set"
SavedMouseClickFirstPastex := "Not Set"
SavedMouseClickFirstPastey := "Not Set"
SavedMouseClickLastPastex := "Not Set"
SavedMouseClickLastPastey := "Not Set"
SavedMouseClickLocation1x := "Not Set"
SavedMouseClickLocation1y := "Not Set"
SavedMouseClickLocation2x := "Not Set"
SavedMouseClickLocation2y := "Not Set"
SavedMouseClickLocation3x := "Not Set"
SavedMouseClickLocation3y := "Not Set"
ImageXRes := 1
ImageYRes := 1



;mouse position toolbar loop
CoordMode, Mouse, Screen
SetTimer, CheckMouse, 20

;gui - interface
gui, Add, GroupBox, h170 w220, Copy Color
gui, Add, Button, xp+10 yp+20 default w150 gGetCopyFirstPixel, Top-Left Pixel
gui, Add, Button, xp+150 default w50 gShowClickCopyFirstPosition, Position
gui, Add, Button, xp-150 yp+25 default w150 gGetCopyLastPixel, Bottom-Right Pixel
gui, Add, Button, xp+150 default w50 gShowClickCopyLastPosition, Position

gui, Add, Text, xp-150 yp+35, Input Image (On Screen) Resolution:
gui, Add, Edit, vImageXRes Limit4 Number w90,
gui, Add, Text, xp+95 w10 Center, X
gui, Add, Edit, xp+15 vImageYRes Limit4 Number w90,

gui, Add, Button, xp-110 yp+35 default w150 gGenerateColorSheet, Get Colors
gui, Add, Button, xp+150 default w50 gCheckColorSheet, Check



gui, Add, GroupBox, xp-160 yp+40 h240 w220, Paste Color
gui, Add, Button, xp+10 yp+20 default w150 gGetPasteFirstPixel, Top-Left Pixel
gui, Add, Button, xp+150 default w50 gShowClickPasteFirstPosition, Position
gui, Add, Button, xp-150 yp+25 default w150 gGetPasteLastPixel, Bottom-Right Pixel
gui, Add, Button, xp+150 default w50 gShowClickPasteLastPosition, Position

gui, Add, Text, xp-150 yp+35, Color Paste Mode:
gui, Add, DropDownList, AltSubmit vPasteMode gSelectPasteMode, RGB|R/G/B Values|R/G/B Buttons|Hex
gui, Add, Button, yp+25 default w150 vlocation1 gloc1, Location
gui, Add, Button, xp+150 default w50 vlocation1p gloc1p, Position
gui, Add, Button, xp-150 yp+25 default w150 vlocation2 gloc2, Location
gui, Add, Button, xp+150 default w50 vlocation2p gloc2p, Position
gui, Add, Button, xp-150 yp+25 default w150 vlocation3 gloc3, Location
gui, Add, Button, xp+150 default w50 vlocation3p gloc3p, Position

gui, Add, Button, xp-150 yp+30 default w200 gPasteColorSheet, Put Colors


gui, Add, GroupBox, xp+220 y7 h380 w220, Options
gui, Add, CheckBox, xp+10 yp+20 vputsomedelay checked%putsomedelay% -Warp w200 gSubmitInfo, Wait 5 Seconds on start
gui, Add, CheckBox, vSkipWhite checked%SkipWhite% -Warp w200 gSubmitInfo, Skip White Color
gui, Add, CheckBox, vSkipBlack checked%SkipBlack% -Warp w200 gSubmitInfo, Skip Black Color
gui, Add, CheckBox, vDragTheMouse checked%DragTheMouse% -Warp w200 gSubmitInfo, Allow Mouse Draging
gui, Add, CheckBox, vDoubleClickPaste checked%DoubleClickPaste% -Warp w200 gSubmitInfo, Double click when pasting colors
gui, Add, CheckBox, vCtrlAPaste checked%CtrlAPaste% -Warp w200 gSubmitInfo, Select All when pasting colors
gui, Add, CheckBox, vSkipColored checked%SkipColored% -Warp w200 gSubmitInfo, Skip Painted Colors
gui, Add, CheckBox, yp+25 vClickBeforeColoring checked%ClickBeforeColoring% w25 gSubmitInfo
gui, Add, Button, yp-5 xp+25 default w175 gGetBeforeColorPixel, Click Before Colors
gui, Add, Text, yp+35 xp-25, Mouse Movement Delay (ms):
gui, Add, Edit, vMouseMoveDelay Limit5 w200 gSubmitInfo, 10
gui, Add, Text, , Mouse Click Delay (ms):
gui, Add, Edit, vMouseClickDelay Limit5 Number w200 gSubmitInfo, 120
gui, Add, Text, , Moving to Color Delay (ms):
gui, Add, Edit, vColorMoveDelay Limit5 w200 gSubmitInfo, 5
gui, Add, Text, , Setting Color Delay (ms):
gui, Add, Edit, vColorDelay Limit5 Number w200 gSubmitInfo, 100


gui, Add, Button, y+20 w80 default gGuiClose, Close

GuiControl, Disable, location1
GuiControl, Disable, location1p
GuiControl, Disable, location2
GuiControl, Disable, location2p
GuiControl, Disable, location3
GuiControl, Disable, location3p

gui, Show, Center, Color Pasting Tool
Tooltip
return


;;;;;;;;


SelectPasteMode:
	gui, Submit, NoHide
	if (PasteMode = 1) {
		GuiControl, Enable, location1
		GuiControl, Enable, location1p
		GuiControl, Disable, location2
		GuiControl, Disable, location2p
		GuiControl, Disable, location3
		GuiControl, Disable, location3p
	} else if (PasteMode = 2) {
		GuiControl, Enable, location1
		GuiControl, Enable, location1p
		GuiControl, Enable, location2
		GuiControl, Enable, location2p
		GuiControl, Enable, location3
		GuiControl, Enable, location3p
	} else if (PasteMode = 3) {
		GuiControl, Enable, location1
		GuiControl, Enable, location1p
		GuiControl, Enable, location2
		GuiControl, Enable, location2p
		GuiControl, Enable, location3
		GuiControl, Enable, location3p
	} else if (PasteMode = 4) {
		GuiControl, Enable, location1
		GuiControl, Enable, location1p
		GuiControl, Disable, location2
		GuiControl, Disable, location2p
		GuiControl, Disable, location3
		GuiControl, Disable, location3p
	} else {
		GuiControl, Disable, location1
		GuiControl, Disable, location1p
		GuiControl, Disable, location2
		GuiControl, Disable, location2p
		GuiControl, Disable, location3
		GuiControl, Disable, location3p
	}
return


;;;;;;;;


SubmitInfo:
	gui, Submit, NoHide
return

loc1:
	ClickToSave := "Location1"
	ShowMousePosition := 1
return

loc2:
	ClickToSave := "Location2"
	ShowMousePosition := 1
return

loc3:
	ClickToSave := "Location3"
	ShowMousePosition := 1
return

loc1p:
	MsgBox, Location Position %SavedMouseClickLocation1x% x %SavedMouseClickLocation1y%
return

loc2p:
	MsgBox, Location Position %SavedMouseClickLocation2x% x %SavedMouseClickLocation2y%
return

loc3p:
	MsgBox, Location Position %SavedMouseClickLocation3x% x %SavedMouseClickLocation3y%
return


GetCopyFirstPixel:
	ClickToSave := "FirstCopy"
	ShowMousePosition := 1
return

ShowClickCopyFirstPosition:
	MsgBox, Top-Left Pixel Position %SavedMouseClickFirstCopyx% x %SavedMouseClickFirstCopyy%
return

GetCopyLastPixel:
	ClickToSave := "LastCopy"
	ShowMousePosition := 1
return

ShowClickCopyLastPosition:
	MsgBox, Top-Left Pixel Position %SavedMouseClickLastCopyx% x %SavedMouseClickLastCopyy%
return

;;;;;;;;


GetPasteFirstPixel:
	ClickToSave := "FirstPaste"
	ShowMousePosition := 1
return

ShowClickPasteFirstPosition:
	MsgBox, Top-Left Pixel Position %SavedMouseClickFirstPastex% x %SavedMouseClickFirstPastey%
return

GetPasteLastPixel:
	ClickToSave := "LastPaste"
	ShowMousePosition := 1
return

ShowClickPasteLastPosition:
	MsgBox, Top-Left Pixel Position %SavedMouseClickLastPastex% x %SavedMouseClickLastPastey%
return

GetBeforeColorPixel:
	ClickToSave := "BeforeColor"
	ShowMousePosition := 1
return

;;;;;;;;


GenerateColorSheet:
	gui, Submit, NoHide
	if(ImageXRes > 0 && ImageYRes > 0) {
		if(SavedMouseClickFirstCopyx != "Not Set"
		 &&SavedMouseClickFirstCopyy != "Not Set"
		 &&SavedMouseClickLastCopyx != "Not Set"
		 &&SavedMouseClickLastCopyy != "Not Set") {

			ImageXDifference := SavedMouseClickLastCopyx - SavedMouseClickFirstCopyx
			ImageYDifference := SavedMouseClickLastCopyy - SavedMouseClickFirstCopyy

			if (ImageXRes <= ImageXDifference && ImageYRes <= ImageYDifference) {
				StoredImage := {}

				updateFastPixelGetColor()

				ImageXIncreasement := ImageXDifference / ImageXRes
				ImageYIncreasement := ImageYDifference / ImageYRes

				RunningCoping := 1

				loop %ImageYRes% {
					LoopOne := A_Index - 1
					PosYRaw := LoopOne * ImageYIncreasement
					PosY := Floor(PosYRaw + SavedMouseClickFirstCopyy + (ImageYIncreasement / 2) + 0.5)
					loop %ImageXRes% {
						LoopTwo := A_Index - 1
						PosXRaw := LoopTwo * ImageXIncreasement
						PosX := Floor(PosXRaw + SavedMouseClickFirstCopyx + (ImageXIncreasement / 2) + 0.5)

						if (RunningCoping = 0) {
							return
						}

						PosColor := fastPixelGetColor(PosX, PosY)
						;MsgBox, %PosX% %PosY% => %PosColor%

						;PixelGetColor, PosColor, PosX, PosY, RGB
						;MouseMove, PosX, PosY

						;Loop % 8 - StrLen(PosColor) {
						;	PosColor := PosColor . "0"
						;}

						;Blue:="0x" SubStr(PosColor,3,2) ;substr is to get the piece
						;Blue:=Blue+0 ;add 0 is to convert it to the current number format

						;Green:="0x" SubStr(PosColor,5,2)
						;Green:=Green+0

						;Red:="0x" SubStr(PosColor,7,2)
						;Red:=Red+0


						Red := PosColor
						Red := Floor(Red - ((Red // 256) * 256))

						Green := ((PosColor - Red) / 256)
						Green := Floor(Green - ((Green // 256) * 256))

						Blue := Floor(((PosColor - Red) / 65536) - (Green / 256))

						if (StoredImage[Red] == "") {
							StoredImage[Red] := {}
						}
						if (StoredImage[Red][Green] == "") {
							StoredImage[Red][Green] := {}
						}
						if (StoredImage[Red][Green][Blue] == "") {
							StoredImage[Red][Green][Blue] := {}
						}
						if (StoredImage[Red][Green][Blue][LoopOne] == "") {
							StoredImage[Red][Green][Blue][LoopOne] := []
						}

						StoredImage[Red][Green][Blue][LoopOne].Push(LoopTwo)
						;MsgBox, % hasValue(StoredImage[Red][Green][Blue][LoopOne], LoopTwo)
						;MsgBox, %PosX%`, %PosY% color %Red% %Green% %Blue% and %PosColor%
					}
				}
				RunningCoping := 0
				MsgBox, Copied Image
			} else {
				MsgBox, Maximum width is %ImageXDifference%`nMaximum Height is %ImageYDifference%
			}
		} else {
			MsgBox, Make Sure to Select the image corners!
		}
	} else {
		MsgBox, Make Sure to Imput The Resolution!
	}
return

CheckColorSheet:
	pixels := 0
	For red, redo in StoredImage {
		For green, greeno in redo {
			For blue, blueo in greeno {
				For posx, posxo in blueo {
					For posy, posyo in posxo {
						pixels++
					}
				}
			}
		}
	}
	MsgBox, There is %pixels% Pixels!
return

GenerateCheckSheet() {
	global
	StoredImageNotEmpity := 0
	For red, redo in StoredImage {
		StoredImageNotEmpity := 1
	}
	if(StoredImageNotEmpity > 0) {
		if(ImageXRes > 0 && ImageYRes > 0) {
			if(SavedMouseClickFirstPastex != "Not Set"
			 &&SavedMouseClickFirstPastey != "Not Set"
			 &&SavedMouseClickLastPastex != "Not Set"
			 &&SavedMouseClickLastPastey != "Not Set") {

				ImageXDifference := SavedMouseClickLastPastex - SavedMouseClickFirstPastex
				ImageYDifference := SavedMouseClickLastPastey - SavedMouseClickFirstPastey

				if (ImageXRes <= ImageXDifference && ImageYRes <= ImageYDifference) {
					CheckedImage := {}

					updateFastPixelGetColor()

					ImageXIncreasement := ImageXDifference / ImageXRes
					ImageYIncreasement := ImageYDifference / ImageYRes

					RunningChecking := 1

					loop %ImageYRes% {
						LoopOne := A_Index - 1
						PosYRaw := LoopOne * ImageYIncreasement
						PosY := Floor(PosYRaw + SavedMouseClickFirstCopyy + (ImageYIncreasement / 2) + 0.5)
						loop %ImageXRes% {
							LoopTwo := A_Index - 1
							PosXRaw := LoopTwo * ImageXIncreasement
							PosX := Floor(PosXRaw + SavedMouseClickFirstCopyx + (ImageXIncreasement / 2) + 0.5)

							if (RunningChecking = 0) {
								return
							}

							PosColor := fastPixelGetColor(PosX, PosY)
							;MsgBox, %PosX% %PosY% => %PosColor%

							;PixelGetColor, PosColor, PosX, PosY, RGB
							;MouseMove, PosX, PosY

							;Loop % 8 - StrLen(PosColor) {
							;	PosColor := PosColor . "0"
							;}

							;Blue:="0x" SubStr(PosColor,3,2) ;substr is to get the piece
							;Blue:=Blue+0 ;add 0 is to convert it to the current number format

							;Green:="0x" SubStr(PosColor,5,2)
							;Green:=Green+0

							;Red:="0x" SubStr(PosColor,7,2)
							;Red:=Red+0


							Red := PosColor
							Red := Floor(Red - ((Red // 256) * 256))

							Green := ((PosColor - Red) / 256)
							Green := Floor(Green - ((Green // 256) * 256))

							Blue := Floor(((PosColor - Red) / 65536) - (Green / 256))

							if (StoredImage[Red] != "" && StoredImage[Red][Green] != "" && StoredImage[Red][Green][Blue] != "" && StoredImage[Red][Green][Blue][LoopOne] != "" && hasValue(StoredImage[Red][Green][Blue][LoopOne], LoopTwo) != 1) {

								;MsgBox, % LoopOne . " and " . LoopTwo

								if (CheckedImage[LoopOne] == "") {
									CheckedImage[LoopOne] := {}
								}

								CheckedImage[LoopOne][LoopTwo] := 1
								;MsgBox, %PosX%`, %PosY% color %Red% %Green% %Blue% and %PosColor%
							}

							
						}
					}
					RunningChecking := 0
				} else {
					MsgBox, Maximum width is %ImageXDifference%`nMaximum Height is %ImageYDifference%
				}
			} else {
				MsgBox, Make Sure to Select the image corners!
			}
		} else {
			MsgBox, Make Sure to Imput The Resolution!
		}
	} else {
		MsgBox, Copy an Image To Check It!
	}
return
}

PasteColorSheet:
	gui, Submit, NoHide
	StoredImageNotEmpity := 0
	For red, redo in StoredImage {
		StoredImageNotEmpity := 1
	}
	if(StoredImageNotEmpity > 0) {
		if(SavedMouseClickFirstPastex != "Not Set"
		 &&SavedMouseClickFirstPastey != "Not Set"
		 &&SavedMouseClickLastPastex != "Not Set"
		 &&SavedMouseClickLastPastey != "Not Set") {

			ImageXDifference := SavedMouseClickLastPastex - SavedMouseClickFirstPastex
			ImageYDifference := SavedMouseClickLastPastey - SavedMouseClickFirstPastey

			if (ImageXRes <= ImageXDifference && ImageYRes <= ImageYDifference) {

				ImageXIncreasement := ImageXDifference / ImageXRes
				ImageYIncreasement := ImageYDifference / ImageYRes

				RunningPasting := 1

				if (putsomedelay = 1) {
					MsgBox, Will Start after clicking OK in 5 seconds!
					Sleep, 5000
				}

				if (SkipColored == 1) {
					GenerateCheckSheet()
				}

				For redvalue, redobj in StoredImage {
					if (PasteMode = 2) {
						if (ClickBeforeColoring = 1) {
							MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
							Sleep, %ColorDelay%
						}

						MouseMove, SavedMouseClickLocation1x - 1, SavedMouseClickLocation1y - 1, %ColorMoveDelay%
						Sleep, %ColorDelay%

						if (DoubleClickPaste = 1) {
							MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 2, 0
						}
						if (CtrlAPaste = 1) {
							MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 1, 0
							Sleep, %ColorDelay%
							Send, ^a
						}

						Sleep, %ColorDelay%

						Send, %redvalue%
						Sleep, %ColorDelay%
					}

					For greenvalue, greenobj in redobj {
						if (PasteMode = 2) {
							if (ClickBeforeColoring = 1) {
								MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
								Sleep, %ColorDelay%
							}

							MouseMove, SavedMouseClickLocation2x - 1, SavedMouseClickLocation2y - 1, %ColorMoveDelay%
							Sleep, %ColorDelay%

							if (DoubleClickPaste = 1) {
								MouseClick, Left, SavedMouseClickLocation2x, SavedMouseClickLocation2y, 2, 0
							}
							if (CtrlAPaste = 1) {
								MouseClick, Left, SavedMouseClickLocation2x, SavedMouseClickLocation2y, 1, 0
								Sleep, %ColorDelay%
								Send, ^a
							}

							Sleep, %ColorDelay%

							Send, %greenvalue%
							Sleep, %ColorDelay%
						}

						For bluevalue, blueobj in greenobj {

							if (SkipWhite = 1 && redvalue = 255 && greenvalue = 255 && bluevalue = 255) {
								continue
							}
							if (SkipBlack = 1 && redvalue = 0 && greenvalue = 0 && bluevalue = 0) {
								continue
							}

							if (PasteMode = 2) {
								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}

								MouseMove, SavedMouseClickLocation3x - 1, SavedMouseClickLocation3y - 1, %ColorMoveDelay%
								Sleep, %ColorDelay%

								if (DoubleClickPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation3x, SavedMouseClickLocation3y, 2, 0
								}
								if (CtrlAPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation3x, SavedMouseClickLocation3y, 1, 0
									Sleep, %ColorDelay%
									Send, ^a
								}

								Sleep, %ColorDelay%

								Send, %bluevalue%
								Sleep, %ColorDelay%
							}
							if (PasteMode = 1) {
								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}

								MouseMove, SavedMouseClickLocation1x - 1, SavedMouseClickLocation1y - 1, %ColorMoveDelay%
								Sleep, %ColorDelay%

								if (DoubleClickPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 2, 0
								}
								if (CtrlAPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 1, 0
									Sleep, %ColorDelay%
									Send, ^a
								}

								Sleep, %ColorDelay%

								Send, %redvalue%`,%greenvalue%`,%bluevalue%
								Sleep, %ColorDelay%

								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}
							}
							if (PasteMode = 3) {
								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}

								topnumber := 1
								if (redvalue < greenvalue) {
									topnumber := 2
								}
								if (bluevalue > redvalue && bluevalue > greenvalue) {
									topnumber := 3
								}
								MouseMove, SavedMouseClickLocation%topnumber%x - 1, SavedMouseClickLocation%topnumber%y - 1, %ColorMoveDelay%
								Sleep, %ColorDelay%
								MouseClick, Left, SavedMouseClickLocation%topnumber%x, SavedMouseClickLocation%topnumber%y, 1, 0
								Sleep, %ColorDelay%
							}
							if (PasteMode = 4) {
								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}

								MouseMove, SavedMouseClickLocation1x - 1, SavedMouseClickLocation1y - 1, %ColorMoveDelay%
								Sleep, %ColorDelay%

								if (DoubleClickPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 2, 0
								}
								if (CtrlAPaste = 1) {
									MouseClick, Left, SavedMouseClickLocation1x, SavedMouseClickLocation1y, 1, 0
									Sleep, %ColorDelay%
									Send, ^a
								}

								Sleep, %ColorDelay%

								Send, % format("{1:02x}{2:02x}{3:02x}", redvalue, greenvalue, bluevalue)
								Sleep, %ColorDelay%

								if (ClickBeforeColoring = 1) {
									MouseClick, Left, SavedMouseClickBeforeColorx, SavedMouseClickBeforeColory, 1, %ColorMoveDelay%
									Sleep, %ColorDelay%
								}
							}

							For posyvalue, posyobj in blueobj {
								PosY := Floor((posyvalue * ImageYIncreasement) + SavedMouseClickFirstPastey + (ImageYIncreasement / 2) + 0.5)

								For posxvalue, posxobj in posyobj {

									if (RunningPasting == 0) {
										return
									}

									PosX := Floor((posxobj * ImageXIncreasement) + SavedMouseClickFirstPastex + (ImageXIncreasement / 2) + 0.5)
									continuethepixel := 1

									if (SkipColored == 1 && CheckedImage[posyvalue] != "" && CheckedImage[posyvalue][posxobj] == 1) {
										;MsgBox, Skipped Something
										continuethepixel := 0
									}

									if (DragTheMouse == 1 && continuethepixel == 1) {

										if (PrevClickY != PosY || (PrevClickXNum + 1) != posxobj) {

											if (PrevClickY != "" && PrevClickX != "") {

												MouseMove, %PrevClickX%, %PrevClickY%, %MouseMoveDelay%
												Send, {LButton Up}
												Sleep, %MouseClickDelay%

											}

											MouseMove, %PosX%, %PosY%, %MouseMoveDelay%
											Send, {LButton Down}
											Sleep, %MouseClickDelay%

										} else {
											MouseMove, %PosX%, %PosY%, %MouseMoveDelay%
										}

										PrevClickY := PosY
										PrevClickX := PosX
										PrevClickXNum := posxobj

									} else if (continuethepixel == 1) {

										;MsgBox, %posxvalue%`, %posyvalue% to %PosX%`, %PosY% color %redvalue% %greenvalue% %bluevalue% and i want %MouseMoveDelay%

										MouseMove, PosX - 1, PosY - 1, %MouseMoveDelay%
										Sleep, %MouseClickDelay%
										;MsgBox, %posxobj% to %PosX% and %posyvalue% to %PosY%
										MouseClick, Left, PosX, PosY, 1, 0
										Sleep, %MouseClickDelay%
									}
								}
							}
						}
					}
				}
				RunningPasting := 0
				MsgBox, Pasted Image
			} else {
				MsgBox, Maximum width is %ImageXDifference%`nMaximum Height is %ImageYDifference%
			}
		} else {
			MsgBox, Make Sure to Select the area to paste!
		}
	} else {
		MsgBox, Copy an Image First!
	}
return



;store the mouse position
~LButton::
	if (ShowMousePosition = 1) {
		MouseGetPos, xx, yy
		SavedMouseClick%ClickToSave%x := xx
		SavedMouseClick%ClickToSave%y := yy
		ShowMousePosition := 0
		;MsgBox, Position Set %xx%`, %yy%
		WinActivate, Color Pasting Tool
	}
return

~Left::
	if (ShowMousePosition = 1) {
		MouseMove, -1, 0, 0, R
	}
return

~+Left::
	if (ShowMousePosition = 1) {
		MouseMove, -5, 0, 0, R
	}
return

~Right::
	if (ShowMousePosition = 1) {
		MouseMove, 1, 0, 0, R
	}
return

~+Right::
	if (ShowMousePosition = 1) {
		MouseMove, 5, 0, 0, R
	}
return

~Up::
	if (ShowMousePosition = 1) {
		MouseMove, 0, -1, 0, R
	}
return

~+Up::
	if (ShowMousePosition = 1) {
		MouseMove, 0, -5, 0, R
	}
return

~Down::
	if (ShowMousePosition = 1) {
		MouseMove, 0, 1, 0, R
	}
return

~+Down::
	if (ShowMousePosition = 1) {
		MouseMove, 0, 5, 0, R
	}
return

~Escape::
	if (RunningPasting = 1) {
		RunningPasting := 0
	} else if (RunningCoping = 1) {
		RunningCoping := 0
	} else if (RunningChecking = 1) {
		RunningChecking := 0
	} else {
		ExitApp
	}
return

;mouse position toolbar
CheckMouse:
	if (ShowMousePosition = 1) {
		MouseGetPos, xx, yy
		Tooltip %xx%`, %yy%
	} else {
		Tooltip
	}
return

GuiClose:
	gui, Destroy
	ExitApp
return

hasValue(haystack, needle) {
    if(!isObject(haystack))
        return false
    if(haystack.Length()==0)
        return false
    for k,v in haystack
        if(v==needle)
            return true
    return false
}
