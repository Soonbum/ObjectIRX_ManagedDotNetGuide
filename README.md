# CADian Managed .NET API (ObjectIRX)를 위한 Developer 가이드

## 개발 환경 설정 방법

* Visual Studio 2022로 작업하였습니다.
  - CADian 플러그인을 개발하려면 "Class Library" 템플릿을 선택해야 하며 .NET Framework 4.8을 선택해야 합니다.
  - 참조 모듈로 다음을 추가하십시오: (1) IcCoreMgd_23.12_16.dll (2) IcMgd_23.12_16.dll (3) TD_Mgd_23.12_16.dll

## 템플릿

### 자동 실행

* APPLOAD 시에 자동으로 실행, 종료되게 하려면 다음과 같이 IExtensionApplication 상속이 필요합니다.

```cs
using IntelliCAD.ApplicationServices;

namespace ExProject
{
    public class MyClass : IExtensionApplication
    {
        public void Initialize()
        {
            Document doc = Application.DocumentManager.MdiActiveDocument;
            doc.Editor.WriteMessage("\nMyClass Initialized");
        }

        public void Terminate()
        {
            Document doc = Application.DocumentManager.MdiActiveDocument;
            doc.Editor.WriteMessage("\nMyClass Terminated");
        }
    }
}
```

### CommandMehod 애트리뷰트

* 함수 앞에 CommandMethod 애트리뷰트를 붙이면 캐드 상에서 커맨드로 함수를 즉시 호출 가능합니다.
  - 아래의 경우 "IterateLayers" 키워드를 통해 캐드에서 함수를 즉각 호출할 수 있습니다.

```cs
// 레이어 목록 출력
[CommandMethod("IterateLayers")]
public static void IterateLayers()
{
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // 현재 데이터베이스의 레이어 테이블 반환
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId,
                                     OpenMode.ForRead) as LayerTable;

        // 레이어 테이블에 있는 레이어 목록을 보여줌
        foreach (ObjectId acObjId in acLyrTbl)
        {
            LayerTableRecord acLyrTblRec;
            acLyrTblRec = acTrans.GetObject(acObjId,
                                            OpenMode.ForRead) as LayerTableRecord;

            acDoc.Editor.WriteMessage("\n" + acLyrTblRec.Name);
        }

        // Dispose of the transaction
    }
}
```

* CommandMethod 애트리뷰트의 flag 값은 다음과 같습니다.
  - 기본 문법은 다음과 같습니다.
    ```cs
    [CommandMethod("CheckForPickfirstSelection", CommandFlags.UsePickSet)]
    public static void CheckForPickfirstSelection()
    {
        ...
    }
    ```

| Flag 값 | 설명 |
| --- | --- |
| ActionMacro |	Command can be recorded as an action with the Action Recorder. |
| Defun	| Command can be invoked as a LISP function and can therefore use acedGetArgs() to receive arguments from LISP and can use the acedRetXxx() functions to return values to LISP. This flag can only be set by the Visual LISP engine. |
| DocExclusiveLock | Document will be exclusively locked when command is invoked. |
| DocReadLock | Document will be read locked when command is invoked. |
| Interruptible	| The command may be interrupted when prompting for user input. |
| Modal	| Command cannot be invoked while another command is active. |
| NoActionRecording	| Command cannot be recorded as action with the Action Recorder. |
| NoBlockEditor	| Command cannot be used from the Block Editor. |
| NoHistory	| Command is not added to the repeat-last-command history list. |
| NoInferConstraint	| Command cannot be used when inferring constraints. |
| NoInternalLock | Document cannot be internally locked. |
| NoMultiple | Command does not support the multiple behavior when prefixed with an astericks (*) as part of a command macro. |
| NoNewStack | Command does not create a new item on the stack. |
| NoOEM | Command cannot be accessed from AutoCAD OEM. |
| NoPaperSpace | Command cannot be used from Paper space. |
| NoPerspective | Command cannot be used when PERSPECTIVE is set to 1. |
| NoTileMode | Command cannot be used when TILEMODE is set to 1. |
| NoUndoMarker | Command does not support undo markers. This is intended for commands that do not modify the database, and therefore should not show up in the undo file. |
| Redraw | When the pickfirst set or grip set are retrieved, they are not cleared. |
| Session | Command is executed in the context of the application rather than the current document context. |
| TempShowDynDimension | Command allows the display of dynamic dimensions temporarily when entities are selected. |
| Transparent | Command can be used while another command is active. |
| Undefined | Command can only be used via its Global Name. |
| UsePickSet | When the pickfirst set is retrieved, it is cleared. |

### 함수 템플릿

* 기본적인 함수의 구조는 다음을 따릅니다.

```cs
[CommandMethod("CommandName")]
public static void CommandName()
{
    Document acDoc = Application.DocumentManager.MdiActiveDocument;    // 도큐먼트
    Database acCurDb = acDoc.Database;                                 // 도큐먼트 내 데이터베이스 (오브젝트, 레이어 등 도면과 관련된 모든 정보 포함)

    // 트랜잭션: DB에서 말하는 그 "트랜잭션"을 의미함
    // Undo/Redo 동작을 위해 동작을 트랜잭션 단위별로 나누어 놓기 위해 필요함
    // 데이터베이스 내 변경사항이 없을 경우 생략해도 됨

    // 만약 내부 스코프의 Open()된 오브젝트들을 한꺼번에 자동으로 Close()하고 싶으면 StartTransaction 대신 StartOpenCloseTransaction 메서드를 사용하면 편리함
    // TransactionManager.NumberOfActiveTransactions 프로퍼티: 활성 트랜잭션 개수를 알 수 있음
    // 만약 Model space에 오브젝트 쓰기 동작을 한 경우, AppendEntity와 AddNewlyCreatedDBObject 함수를 사용해야 함
    // Nested 구조를 가질 수도 있음
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // TODO

        acTrans.Commit();    // DB에 변경사항 반영 (트랜잭션 사용시에만 해당됨), 만약 .Abort() 함수를 사용하면 변경사항을 버림 (롤백)
    }   // using 문을 사용하면 자동으로 acTrans.Dispose()를 호출함
}
```

### 계층 구조

* 먼저 모듈은 다음과 같은 구조로 되어 있습니다. (자주 사용하는 모듈 위주로만 기록되어 있음)

* Application
  - DocumentManager
    * Document
      - Editor
        * ViewTable
          - ViewTableRecord
        * ViewportTable
          - ViewportTableRecord
      - Database
        * BlockTable
          - BlockTableRecord
        * LayerTable
          - LayerTableRecord

### 오브젝트

* 오브젝트의 ID는 다음과 같습니다.
  - Entity handle: 이 ID는 불변입니다. 도면을 외부 파일로 내보낸 뒤에라도 나중에 객체에 접근할 때 유용한 방법입니다.
  - ObjectId: DB가 메모리에 로드된 동안에만 유효합니다. 휘발성이 있는 ID입니다. (가장 많이 사용하는 방법)
  - Instance pointer
* GetObject 메서드는 DBObject를 리턴합니다.

### 기능 구현 예시

* 레이어 관리
  - 레이어 추가
    ```cs
    // 현재 데이터베이스의 레이어 테이블 반환
    LayerTable acLyrTbl;
    acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

    // 레이어 "MyLayer"가 존재하지 않으면
    if (acLyrTbl.Has("MyLayer") != true)
    {
        // 쓰기 작업을 위한 레이어 테이블 열기
        acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

        // 레이어 "MyLayer"라는 새로운 레이어 테이블 레코드 생성
        using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
        {
            acLyrTblRec.Name = "MyLayer";

            // 레이어 테이블과 트랜잭션에 새로운 레이어 테이블 레코드 추가
            acLyrTbl.Add(acLyrTblRec);
            acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
        }

        // Commit the changes
        acTrans.Commit();
    }
    ```
  - 레이어 조회
    ```cs
    // 현재 데이터베이스의 레이어 테이블 반환
    LayerTable acLyrTbl;
    acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

    // 레이어 테이블에 있는 레이어 목록을 보여줌
    foreach (ObjectId acObjId in acLyrTbl)
    {
        LayerTableRecord acLyrTblRec;
        acLyrTblRec = acTrans.GetObject(acObjId, OpenMode.ForRead) as LayerTableRecord;

        acDoc.Editor.WriteMessage("\n" + acLyrTblRec.Name);
    }
    ```
  - 레이어 찾기
    ```cs
    // 현재 데이터베이스의 레이어 테이블 반환
    LayerTable acLyrTbl;
    acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

    // 레이어 테이블에서 레이어 "MyLayer"가 존재하는지 확인
    if (acLyrTbl.Has("MyLayer") != true)
    {
        acDoc.Editor.WriteMessage("\n'MyLayer' does not exist");
    }
    else
    {
        acDoc.Editor.WriteMessage("\n'MyLayer' exists");
    }
    ```
  - 레이어 삭제
    ```cs
    // 현재 데이터베이스의 레이어 테이블 반환
    LayerTable acLyrTbl;
    acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

    // 레이어 테이블에서 레이어 "MyLayer"가 존재하는지 확인
    if (acLyrTbl.Has("MyLayer") == true)
    {
        LayerTableRecord acLyrTblRec;
        acLyrTblRec = acTrans.GetObject(acLyrTbl["MyLayer"], OpenMode.ForWrite) as LayerTableRecord;

        try
        {
            acLyrTblRec.Erase();
            acDoc.Editor.WriteMessage("\n'MyLayer' was erased");

            // Commit the changes
            acTrans.Commit();
        }
        catch
        {
            acDoc.Editor.WriteMessage("\n'MyLayer' could not be erased");
        }
    }
    else
    {
        acDoc.Editor.WriteMessage("\n'MyLayer' does not exist");
    }
    ```

* 도큐먼트 창 업데이트

```cs
// Redraw the drawing
Application.UpdateScreen();
Application.DocumentManager.MdiActiveDocument.Editor.UpdateScreen();
 
// Regenerate the drawing
Application.DocumentManager.MdiActiveDocument.Editor.Regen();
```

* 사각형 영역 마우스로 입력 받고 이미지 파일로 플롯(인쇄)하기

```cs
Document doc = IntelliCAD.ApplicationServices.Application.DocumentManager.MdiActiveDocument;
Editor ed = doc.Editor;
Database db = doc.Database;

// 사용자로부터 두 점을 입력받아 화면 범위를 선택
PromptPointResult ppr1 = ed.GetPoint("첫 번째 지점을 선택하십시오:");

if (ppr1.Status != PromptStatus.OK) return;
Point3d firstPoint = ppr1.Value;

if (ppr1.Status == PromptStatus.Cancel) return;

PromptPointResult ppr2 = ed.GetCorner("두 번째 지점을 선택하십시오:", ppr1.Value);

if (ppr2.Status != PromptStatus.OK) return;
Point3d secondPoint = ppr2.Value;

// 선택된 두 점을 통해서 Extents2d 객체 생성
Extents2d plotWindow = new Extents2d(
    Math.Min(firstPoint.X, secondPoint.X),
    Math.Min(firstPoint.Y, secondPoint.Y),
    Math.Max(firstPoint.X, secondPoint.X),
    Math.Max(firstPoint.Y, secondPoint.Y)
);

using (Transaction tr = db.TransactionManager.StartTransaction())
{
    // 용지 방향
    bool OrientationLandScape = true;

    // 팝업 컨트롤 값에 따라 용지 방향 결정
    if (popupContainerEdit_PaperOrientation.Text == "가로")
        OrientationLandScape = true;
    else
        OrientationLandScape = false;

    // 용지 크기
    string paperSize = popupContainerEdit_PaperSize.Text;

    // 파일 저장 다이얼로그 표시
    System.Windows.Forms.SaveFileDialog saveFileDialog = new System.Windows.Forms.SaveFileDialog();
    saveFileDialog.Filter = "이미지 파일|*.png";
    saveFileDialog.Title = "이미지 파일을 저장하십시오.";
    saveFileDialog.FileName = "TestPrint.png";

    if (saveFileDialog.ShowDialog() == DialogResult.OK)
    {
        string outputFilePath = saveFileDialog.FileName;

        // 현재 레이아웃을 플롯(인쇄)
        BlockTableRecord currentSpace = (BlockTableRecord)tr.GetObject(db.CurrentSpaceId, OpenMode.ForRead);
        Layout layout = (Layout)tr.GetObject(currentSpace.LayoutId, OpenMode.ForRead);

        // PlotInfo 객체를 레이아웃에 연결
        PlotInfo plotInfo = new PlotInfo();
        plotInfo.Layout = currentSpace.LayoutId;

        // 레이아웃 설정을 기반으로 PlotSettings 객체 설정
        PlotSettings plotSettings = new PlotSettings(layout.ModelType);
        plotSettings.CopyFrom(layout);

        // The PlotSettingsValidator helps create a valid PlotSettings object
        PlotSettingsValidator plotSettingsValidator = PlotSettingsValidator.Current;
        plotSettingsValidator.SetPlotWindowArea(plotSettings, plotWindow);      // 화면 범위

        // 화면 맞춤
        plotSettingsValidator.SetPlotType(plotSettings, PlotType.Window);    // 인쇄 범위 > 인쇄 대상: 윈도우
        plotSettingsValidator.SetUseStandardScale(plotSettings, true);
        plotSettingsValidator.SetStdScaleType(plotSettings, StdScaleType.ScaleToFit);
        plotSettingsValidator.SetPlotCentered(plotSettings, true);
        if (OrientationLandScape)
            plotSettingsValidator.SetPlotRotation(plotSettings, PlotRotation.Degrees090);   // 가로 방향
        else
            plotSettingsValidator.SetPlotRotation(plotSettings, PlotRotation.Degrees000);   // 세로 방향

        // Use standard DWF PC3, as for today we're just plotting to file
        plotSettingsValidator.SetPlotConfigurationName(plotSettings, "PublishToWeb PNG.pc3", paperSize); // Needs to be a device that can print directly and valid media name.
        // plotSettingsValidator.SetCurrentStyleSheet(plotSettings, "Icad.ctb");    // Set stylesheet

        // Save the modified PlotSettings to the database
        layout.UpgradeOpen();
        layout.CopyFrom(plotSettings);

        plotInfo.OverrideSettings = plotSettings;
        PlotInfoValidator plotInfoValidator = new PlotInfoValidator();
        plotInfoValidator.MediaMatchingPolicy = MatchingPolicy.MatchEnabled;
        plotInfoValidator.Validate(plotInfo);

        // A PlotEngine does the actual plotting
        if (PlotFactory.ProcessPlotState == ProcessPlotState.NotPlotting)
        {
            try
            {
                PlotEngine plotEngine = PlotFactory.CreatePublishEngine();

                plotEngine.BeginPlot(null, null);
                PlotConfig config = plotInfo.ValidatedConfig;
                config.IsPlotToFile = true;
                plotEngine.BeginDocument(plotInfo, "TestPrint", null, 1, true, outputFilePath);
                PlotPageInfo plotPageInfo = new PlotPageInfo();
                plotEngine.BeginPage(plotPageInfo, plotInfo, true, null);
                plotEngine.BeginGenerateGraphics(null);

                plotEngine.EndGenerateGraphics(null);
                plotEngine.EndPage(null);
                plotEngine.EndDocument(null);
                plotEngine.EndPlot(null);
                plotEngine.Destroy();
                plotEngine = null;
            }
            catch (System.Exception error) { System.Exception Err = error; }

            MessageBox.Show("Plot을 완료했습니다.", "성공", MessageBoxButtons.OK, MessageBoxIcon.Information);
        }
        else
        {
            ed.WriteMessage("\n다른 플롯이 진행 중입니다.");
        }
    }

    tr.Commit();
}
```

* 리본 메뉴 생성하기

```cs
public class CuiManager : IDisposable
{
    // 리본 메뉴에 대한 문자열 리터럴
    public static class Literals
    {
        public const string commandCreateRibbonTab = "CadianAICreateRibbonTab";
        public const string commandDeleteRibbonTab = "CadianAIDeleteRibbonTab";
        public const string commandToggleRibbonTab = "CadianAIToggleRibbonTab";

        public const string uidRibbonTab = "dotnet-cadian-ai-tab-1";
        public const string uidRibbonPanel = "dotnet-cadian-ai-panel-1";

        public const string uidCuiElementBase = "dotnet-cadian-ai-{0}-{1}";
        public const string defaultHelpBase = "Help: {0}";
    }

    private List<MenuMacroBase> cuiElements = new List<MenuMacroBase>();
    private static CuiManager manager = null;
    private static bool applsQuitting = false;

    public CuiManager()
    {
        IntelliCAD.ApplicationServices.Application.BeginQuit += Application_BeginQuit;
        IntelliCAD.ApplicationServices.Application.DocumentManager.DocumentToBeDestroyed += DocumentManager_DocumentToBeDestroyed;
    }

    ~CuiManager()
    {
        Dispose();
    }

    public void Dispose()
    {
        if (null != manager)
        {
            foreach (MenuMacroBase cuiElement in cuiElements)
            {
                CleanupCuiElement(cuiElement);
                cuiElement.Dispose();
            }
            cuiElements.Clear();

            manager = null;
        }

        GC.SuppressFinalize(this);
    }

    public static CuiManager GetManager()
    {
        if (null == manager)
        {
            manager = new CuiManager();
        }
        return manager;
    }

    public MenuMacroBase RegisterCuiElement(MenuMacroBase cuiElement)
    {
        if (null == cuiElement)
            throw new ArgumentNullException(nameof(cuiElement));
        if (!cuiElements.Exists(e => e.Uid == cuiElement.Uid))
            cuiElements.Add(cuiElement);

        return cuiElement;
    }

    public void DeleteCuiElement(MenuMacroBase cuiElement)
    {
        if (null == cuiElement)
            throw new ArgumentNullException(nameof(cuiElement));

        MenuMacroBase found = cuiElements.Find(e => e.Uid == cuiElement.Uid);

        if (null != found)
        {
            CleanupCuiElement(found);
            cuiElements.Remove(found);
            found.Dispose();
        }
        else
        {
            CleanupCuiElement(cuiElement);
        }
    }

    private void CleanupCuiElement(MenuMacroBase cuiElement)
    {
        using (CuiProfile profile = new CuiProfile())
        {
            using (Workspace ws = profile.GetDefaultWorkspace())
            {
                if (cuiElement is RibbonTab)
                {
                    ws.Remove(CuiItems.RIBBON, cuiElement.Uid);
                    profile.DeleteRibbonTab(cuiElement.Uid);
                }
                else if (cuiElement is RibbonPanel)
                {
                    ws.Remove(CuiItems.RIBBON, cuiElement.Uid);
                    profile.DeleteRibbonPanel(cuiElement.Uid);
                }
            }
        }
    }

    private static void Application_BeginQuit(object sender, EventArgs e)
    {
        applsQuitting = true;
    }

    private static void DocumentManager_DocumentToBeDestroyed(object sender, DocumentCollectionEventArgs e)
    {
        if (applsQuitting && IntelliCAD.ApplicationServices.Application.DocumentManager.Count == 1)
            manager?.Dispose();
    }

    // 리본 컨트롤 생성하기
    [CommandMethod(CuiManager.Literals.commandCreateRibbonTab)]
    public static void CadianAICreateRibbonTab()
    {
        CuiProfile profile = null;
        RibbonCommandItem button = null;
        RibbonRow row = null;
        RibbonSeparator separator = null;
        RibbonTab tab = null;
        RibbonPanel panel = null;
        Workspace ws = null;
        try
        {
            profile = new CuiProfile();

            if (profile.GetMacroBaseByUid(CuiManager.Literals.uidRibbonPanel) != null
                || profile.GetMacroBaseByUid(CuiManager.Literals.uidRibbonTab) != null)
            {
                return;
            }

            // 1번째 버튼
            button = new RibbonCommandItem();
            button.DefaultToolTip = "Wall Vector Detection";
            button.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, button.DefaultToolTip);
            button.Style = CuiRibbonButtonStyle.LARGE_WITH_TEXT;
            button.BMPFileNameLgColor = @".\RibbonImage\IconSetArrows3_32x32.bmp";
            button.Command = "VectorDetection";
            button.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, button.GetType().Name, 1);

            row = new RibbonRow();
            row.Add(button);
            button.Dispose();

            // 구분자
            separator = new RibbonSeparator();
            separator.Style = CuiRibbonSeparatorStyle.LINE;
            separator.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, separator.GetType().Name, 1);

            row.Add(separator);
            separator.Dispose();

            // 2번째 버튼
            button = new RibbonCommandItem();
            button.DefaultToolTip = "Export Image";
            button.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, button.DefaultToolTip);
            button.Style = CuiRibbonButtonStyle.LARGE_WITH_TEXT;
            button.BMPFileNameLgColor = @".\RibbonImage\ExportToIMG_32x32.bmp";
            button.Command = "DWGtoIMG";
            button.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, button.GetType().Name, 1);

            row.Add(button);
            button.Dispose();

            // 구분자
            separator = new RibbonSeparator();
            separator.Style = CuiRibbonSeparatorStyle.LINE;
            separator.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, separator.GetType().Name, 1);

            row.Add(separator);
            separator.Dispose();

            // 3번째 버튼
            button = new RibbonCommandItem();
            button.DefaultToolTip = "DWG Clipping";
            button.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, button.DefaultToolTip);
            button.Style = CuiRibbonButtonStyle.LARGE_WITH_TEXT;
            //button.BMPFileNameLgColor = @".\RibbonImage\ExportToIMG_32x32.bmp";
            button.Command = "DWGClipping";
            button.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, button.GetType().Name, 1);

            row.Add(button);
            button.Dispose();

            // 구분자
            separator = new RibbonSeparator();
            separator.Style = CuiRibbonSeparatorStyle.LINE;
            separator.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, separator.GetType().Name, 1);

            row.Add(separator);
            separator.Dispose();

            // 4번째 버튼
            button = new RibbonCommandItem();
            button.DefaultToolTip = "Send Images to Server";
            button.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, button.DefaultToolTip);
            button.Style = CuiRibbonButtonStyle.LARGE_WITH_TEXT;
            //button.BMPFileNameLgColor = @".\RibbonImage\ExportToIMG_32x32.bmp";
            button.Command = "SendToServer";
            button.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, button.GetType().Name, 1);

            row.Add(button);
            button.Dispose();

            // 리본 탭, 리본 패널 생성
            tab = new RibbonTab();
            tab.Uid = CuiManager.Literals.uidRibbonTab;
            CuiManager.GetManager().RegisterCuiElement(tab);
            tab.GroupName = "ICAD";
            tab.DefaultText = "캐디안 AI";
            tab.DefaultToolTip = tab.DefaultText;
            tab.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, tab.DefaultToolTip);
            tab.DoCreateTab = true;

            panel = new RibbonPanel();
            panel.Uid = CuiManager.Literals.uidRibbonPanel;
            CuiManager.GetManager().RegisterCuiElement(panel);
            panel.DefaultToolTip = "캐디안 AI";
            panel.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, panel.DefaultToolTip);
            panel.DoCreatePanel = true;

            panel.Insert(row, null);
            tab.Insert(panel, "");

            profile.AddRibbonPanel(panel, tab.Uid);
            profile.InsertRibbonTab(tab, "");

            ws = profile.GetDefaultWorkspace();
            ws.UpdatePanelsFromImport(panel, profile);
            ws.UpdateTabsFromImport(tab, profile);

            profile.BuildUILayout(true, false, false, true);
        }
        finally
        {
            profile?.Dispose();
            button?.Dispose();
            row?.Dispose();
            separator?.Dispose();
            ws?.Dispose();
        }
    }

    // 리본에서 test 탭 제거
    [CommandMethod(CuiManager.Literals.commandDeleteRibbonTab)]
    public static void CadianAIDeleteRibbonTab()
    {
        using (CuiProfile profile = new CuiProfile())
        {
            MenuMacroBase panel = profile.GetMacroBaseByUid(CuiManager.Literals.uidRibbonPanel);
            if (null != panel)
            {
                CuiManager.GetManager().DeleteCuiElement(panel);
            }

            MenuMacroBase tab = profile.GetMacroBaseByUid(CuiManager.Literals.uidRibbonTab);
            if (null != tab)
            {
                CuiManager.GetManager().DeleteCuiElement(tab);
            }

            profile.BuildUILayout(true, false, false, true);
        }
    }

    // 리본 탭 on/off 토글
    [CommandMethod(CuiManager.Literals.commandToggleRibbonTab)]
    public static void CadianAIToggleRibbonTab()
    {
        CuiProfile profile = null;
        IntelliCAD.EditorInput.Editor editor = null;
        try
        {
            profile = new CuiProfile();
            editor = IntelliCAD.ApplicationServices.Application.DocumentManager.MdiActiveDocument.Editor;

            if (profile.GetMacroBaseByUid(CuiManager.Literals.uidRibbonTab) == null)
            {
                editor.Command(CuiManager.Literals.commandCreateRibbonTab);
            }
            else
            {
                editor.Command(CuiManager.Literals.commandDeleteRibbonTab);
            }

            profile.BuildUILayout(true, false, false, true);
        }
        finally
        {
            profile.Dispose();
            editor.Dispose();
        }
    }

    // CUI에서 모든 test 객체를 정리함
    [CommandMethod("ExCuiCleanup")]
    public static void ExCuiCleanup()
    {
        CuiManager.GetManager().Dispose();
        using (CuiProfile profile = new CuiProfile())
        {
            profile.BuildUILayout(true, true, true, true);
        }
    }
}
```

* 시스템 변수 설정

```cs
// Get the current value from a system variable
int nMaxSort = System.Convert.ToInt32(Application.GetSystemVariable("MAXSORT"));

// Set system variable to new value
Application.SetSystemVariable("MAXSORT", 100);
```

* 프롬프트에서 문자열 입력 받기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

PromptStringOptions pStrOpts = new PromptStringOptions("\nEnter your name: ");
pStrOpts.AllowSpaces = true;    // 공백 허용
PromptResult pStrRes = acDoc.Editor.GetString(pStrOpts);    // 사용자에게 문자열 입력 요청

Application.ShowAlertDialog("The name entered was: " + pStrRes.StringResult);
```

* 프롬프트에서 키워드 입력 받기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

PromptKeywordOptions pKeyOpts = new PromptKeywordOptions("");
pKeyOpts.Message = "\nEnter an option ";
pKeyOpts.Keywords.Add("Line");
pKeyOpts.Keywords.Add("Circle");
pKeyOpts.Keywords.Add("Arc");
pKeyOpts.AllowNone = false;    // 무조건 선택을 해야 함

PromptResult pKeyRes = acDoc.Editor.GetKeywords(pKeyOpts);    // 사용자에게 키워드 입력 요청

Application.ShowAlertDialog("Entered keyword: " + pKeyRes.StringResult);
```

* 프롬프트에서 정수, 키워드 혼합 입력 받기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

PromptIntegerOptions pIntOpts = new PromptIntegerOptions("");
pIntOpts.Message = "\nEnter the size or ";

// 0이나 음수는 허용하지 않으며 양수만 입력할 수 있음
pIntOpts.AllowZero = false;
pIntOpts.AllowNegative = false;

// 키워드는 3개를 받을 수 있으며 기본값을 지정함, 키워드를 입력하지 않을 수도 있음
pIntOpts.Keywords.Add("Big");
pIntOpts.Keywords.Add("Small");
pIntOpts.Keywords.Add("Regular");
pIntOpts.Keywords.Default = "Regular";
pIntOpts.AllowNone = true;

// 사용자에게 정수 입력 요청
PromptIntegerResult pIntRes = acDoc.Editor.GetInteger(pIntOpts);

// 키워드인지 아닌지에 따라 다르게 동작
if (pIntRes.Status == PromptStatus.Keyword)
{
    Application.ShowAlertDialog("Entered keyword: " + pIntRes.StringResult);
}
else
{
    Application.ShowAlertDialog("Entered value: " + pIntRes.Value.ToString());
}
```

* 프롬프트에서 명령어 호출하기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

// 마지막의 whitespace는 Enter와 같음
acDoc.SendStringToExecute("._circle 2,2,0 4 ", true, false, false);    // 중심이 (2, 2, 0)이고 반지름이 4인 원
acDoc.SendStringToExecute("._zoom _all ", true, false, false);         // 모든 것이 보이도록 Zoom
```

* 오브젝트 그리기
  - Line 오브젝트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // (5, 5, 0)과 (12, 3, 0)을 잇는 라인 생성
        using (Line acLine = new Line(new Point3d(5, 5, 0), new Point3d(12, 3, 0)))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acLine);
            acTrans.AddNewlyCreatedDBObject(acLine, true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - Polyline 오브젝트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // 3점을 잇는 2개의 세그먼트를 가진 폴리라인 생성
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(2, 4), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(4, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(6, 4), 0, 0, 0);

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - Circle 오브젝트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // 중심 (2, 3, 0), 반지름 4.25인 원 생성
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 3, 0);
            acCirc.Radius = 4.25;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - Arc 오브젝트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // 중심 (6.25, 9.125, 0), 반지름 6, 시작 각도 64 degree, 끝 각도 204 degree인 호 생성
        // (64 degree = 1.117 radian, 204 degree = 3.5605)
        using (Arc acArc = new Arc(new Point3d(6.25, 9.125, 0), 6, 1.117, 3.5605))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acArc);
            acTrans.AddNewlyCreatedDBObject(acArc, true);
        }

        // Save the new line to the database
        acTrans.Commit();
    }
    ```
  - Spline 오브젝트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // 스플라인 곡선이 지나갈 점 3개를 지정함
        Point3dCollection ptColl = new Point3dCollection();
        ptColl.Add(new Point3d(0, 0, 0));
        ptColl.Add(new Point3d(5, 5, 0));
        ptColl.Add(new Point3d(10, 0, 0));

        // 점 (0.5, 0.5, 0)의 3D 벡터 가져옴
        Vector3d vecTan = new Point3d(0.5, 0.5, 0).GetAsVector();

        // 시작 및 끝 접선이 (0.5, 0.5, 0.0)이고 점 (0, 0, 0), (5, 5, 0), (10, 0, 0)을 통과하는 스플라인을 생성함
        using (Spline acSpline = new Spline(ptColl, vecTan, vecTan, 4, 0.0))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSpline);
            acTrans.AddNewlyCreatedDBObject(acSpline, true);
        }

        // Save the new line to the database
        acTrans.Commit();
    }
    ```
  - Point 오브젝트
    * Pdmode: 점 스타일 지정 (Pdmode == 1은 안 보이는 점)
      - ![image](https://github.com/Soonbum/ObjectIRX_ManagedDotNetGuide/assets/16474083/1046474b-b3c8-4f07-b6ed-39cc26fb34ce)
      - ![image](https://github.com/Soonbum/ObjectIRX_ManagedDotNetGuide/assets/16474083/de868930-4172-4224-8839-6019dfb992a0)
    * Pdsize: 점 크기 지정 (Pdmode가 0, 1 이외인 경우)
      - 0: 그래픽 영역 높이의 5% 크기
      - 양수: 포인트 이미지의 크기
      - 음수: 뷰포트 크기의 퍼센트
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Model space의 (4, 3, 0)에 점 생성
        using (DBPoint acPoint = new DBPoint(new Point3d(4, 3, 0)))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoint);
            acTrans.AddNewlyCreatedDBObject(acPoint, true);
        }

        // 도면의 모든 점 오브젝트에 대한 스타일 설정
        acCurDb.Pdmode = 34;
        acCurDb.Pdsize = 1;

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - Solid-Filled 영역
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // 설명: 처음 두 점은 폴리곤의 가장자리 하나를 정의함, 3번째 점은 2번째 점으로부터 대각선 반대편 점을 의미함, 4번째 점과 3번째 점이 같을 경우 삼각형이 됨

        // Model space에 (나비 넥타이 모양) 사변형 솔리드 생성
        using (Solid ac2DSolidBow = new Solid(new Point3d(0, 0, 0),
                                        new Point3d(5, 0, 0),
                                        new Point3d(5, 8, 0),
                                        new Point3d(0, 8, 0)))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(ac2DSolidBow);
            acTrans.AddNewlyCreatedDBObject(ac2DSolidBow, true);
        }

        // Model space에 (정사각형) 사변형 솔리드 생성
        using (Solid ac2DSolidSqr = new Solid(new Point3d(10, 0, 0),
                                        new Point3d(15, 0, 0),
                                        new Point3d(10, 8, 0),
                                        new Point3d(15, 8, 0)))
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(ac2DSolidSqr);
            acTrans.AddNewlyCreatedDBObject(ac2DSolidSqr, true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```

<!--
https://help.autodesk.com/view/OARX/2024/ENU/?guid=GUID-9CD22AE5-8F66-4925-A155-95852BAFD565
-->
