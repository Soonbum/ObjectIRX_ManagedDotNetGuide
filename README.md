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
Teigha.Geometry.Point3d firstPoint = ppr1.Value;

if (ppr1.Status == PromptStatus.Cancel) return;

PromptPointResult ppr2 = ed.GetCorner("두 번째 지점을 선택하십시오:", ppr1.Value);

if (ppr2.Status != PromptStatus.OK) return;
Teigha.Geometry.Point3d secondPoint = ppr2.Value;

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

        // scaled to fit
        plotSettingsValidator.SetPlotType(plotSettings, PlotType.Window);
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

            // outputFilePath 이미지 파일을 그레이스케일로 변환
            if (checkEdit_grayscale.Checked)
            {
                using (var src = new Mat(outputFilePath, ImreadModes.Color))
                {
                    // 그레이스케일 이미지로 변환
                    using (var gray = new Mat())
                    {
                        Cv2.CvtColor(src, gray, ColorConversionCodes.BGR2GRAY);

                        using (var enhancedGray = new Mat())
                        {
                            // 히스토그램 균등화를 통해 대비를 증가시킴
                            Cv2.EqualizeHist(gray, enhancedGray);

                            // 그레이스케일 이미지 저장
                            Cv2.ImWrite(outputFilePath, enhancedGray);
                        }
                    }
                }
            }

            XtraMessageBox.Show("Plot을 완료했습니다.", "성공", MessageBoxButtons.OK, MessageBoxIcon.Information);
        }

        else
        {
            ed.WriteMessage("\n다른 플롯이 진행 중입니다.");
        }
    }

    tr.Commit();
}
```

* 레이어 별로 이미지 Clipping해서 플롯(인쇄)하기

```cs
private async void ImageSaveButton_Click(object sender, EventArgs e)
{
    Document doc = IntelliCAD.ApplicationServices.Application.DocumentManager.MdiActiveDocument;
    Editor ed = doc.Editor;
    Database db = doc.Database;

    // 사용자로부터 두 점을 입력받아 화면 범위를 선택
    PromptPointResult ppr1 = ed.GetPoint("첫 번째 지점을 선택하십시오:");

    if (ppr1.Status != PromptStatus.OK) return;
    Teigha.Geometry.Point3d firstPoint = ppr1.Value;

    if (ppr1.Status == PromptStatus.Cancel) return;

    PromptPointResult ppr2 = ed.GetCorner("두 번째 지점을 선택하십시오:", ppr1.Value);

    if (ppr2.Status != PromptStatus.OK) return;
    Teigha.Geometry.Point3d secondPoint = ppr2.Value;

    // 선택된 두 점을 통해서 Extents2d 객체 생성
    Extents2d plotWindow = new Extents2d(
        Math.Min(firstPoint.X, secondPoint.X),
        Math.Min(firstPoint.Y, secondPoint.Y),
        Math.Max(firstPoint.X, secondPoint.X),
        Math.Max(firstPoint.Y, secondPoint.Y)
    );

    // 진행바 초기화
    progressBarControl.Position = 0;

    using (Transaction tr = db.TransactionManager.StartTransaction())
    {
        // 용지 방향
        bool OrientationLandScape = true;

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
            progressBarControl.Show();

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

            // scaled to fit
            plotSettingsValidator.SetPlotType(plotSettings, PlotType.Window);
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
                // 전체 플롯
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

                // outputFilePath 이미지 파일을 그레이스케일로 변환
                if (checkEdit_grayscale.Checked)
                {
                    using (var src = new Mat(outputFilePath, ImreadModes.Color))
                    {
                        // 그레이스케일 이미지로 변환
                        using (var gray = new Mat())
                        {
                            Cv2.CvtColor(src, gray, ColorConversionCodes.BGR2GRAY);

                            using (var enhancedGray = new Mat())
                            {
                                // 히스토그램 균등화를 통해 대비를 증가시킴
                                Cv2.EqualizeHist(gray, enhancedGray);

                                // 그레이스케일 이미지 저장
                                Cv2.ImWrite(outputFilePath, enhancedGray);
                            }
                        }
                    }
                }

                // 현재 데이터베이스의 레이어 테이블 반환
                LayerTable acLyrTbl;
                acLyrTbl = tr.GetObject(db.LayerTableId, OpenMode.ForRead) as LayerTable;

                // 플러그인 실행 전 레이어 상태를 저장함
                List<(LayerTableRecord, bool)> layerList = new List<(LayerTableRecord, bool)>();

                foreach (ObjectId acObjId in acLyrTbl)
                {
                    LayerTableRecord acLyrTblRec;
                    acLyrTblRec = tr.GetObject(acObjId, OpenMode.ForWrite) as LayerTableRecord;

                    layerList.Add((acLyrTblRec, acLyrTblRec.IsOff));
                }

                // 레이어 모두 끄기
                foreach (ObjectId acObjId in acLyrTbl)
                {
                    LayerTableRecord acLyrTblRec;
                    acLyrTblRec = tr.GetObject(acObjId, OpenMode.ForWrite) as LayerTableRecord;

                    acLyrTblRec.IsOff = true;
                }

                // 레이어를 하나씩 켜고 플롯 후 레이어를 다시 끔
                foreach (ObjectId acObjId in acLyrTbl)
                {
                    LayerTableRecord acLyrTblRec;
                    acLyrTblRec = tr.GetObject(acObjId, OpenMode.ForWrite) as LayerTableRecord;

                    acLyrTblRec.IsOff = false;  // 레이어 켜기

                    // 만약 acLyrTblRec.Name 안에 파일 이름으로 쓸 수 없는 문자가 있으면 _로 대체
                    string invalidChars = new string(Path.GetInvalidFileNameChars()) + new string(Path.GetInvalidPathChars());
                    string invalidRegex = string.Format("[{0}]", Regex.Escape(invalidChars));
                    string safeName = Regex.Replace(acLyrTblRec.Name, invalidRegex, "_");

                    // 파일명은 "TestPrint - 레이어명.png"로 저장 (TestPrint는 기본값)
                    string newOutputFilePath = outputFilePath.Replace(".png", " - " + safeName + ".png");

                    // 진행바 표시하기
                    progressBarControl.Properties.Step = 1;
                    progressBarControl.Properties.Maximum = layerList.Count();
                    await Task.Run(() =>
                    {
                        lock (progressBarControl)
                        {
                            progressBarControl.PerformStep();
                        }
                    });


                    // 레이어별 플롯
                    try
                    {
                        PlotEngine plotEngine = PlotFactory.CreatePublishEngine();

                        plotEngine.BeginPlot(null, null);
                        PlotConfig config = plotInfo.ValidatedConfig;
                        config.IsPlotToFile = true;
                        plotEngine.BeginDocument(plotInfo, "TestPrint", null, 1, true, newOutputFilePath);
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

                    // newOutputFilePath 이미지 파일을 그레이스케일로 변환
                    if (checkEdit_grayscale.Checked)
                    {
                        using (var src = new Mat(newOutputFilePath, ImreadModes.Color))
                        {
                            // 그레이스케일 이미지로 변환
                            using (var gray = new Mat())
                            {
                                Cv2.CvtColor(src, gray, ColorConversionCodes.BGR2GRAY);

                                using (var enhancedGray = new Mat())
                                {
                                    // 히스토그램 균등화를 통해 대비를 증가시킴
                                    Cv2.EqualizeHist(gray, enhancedGray);

                                    // 그레이스케일 이미지 저장
                                    Cv2.ImWrite(newOutputFilePath, enhancedGray);
                                }
                            }
                        }
                    }

                    // 흰색 이미지인 경우 파일 삭제
                    Mat image = Cv2.ImRead(newOutputFilePath, ImreadModes.Color);
                    if (IsWhiteImage(image))
                    {
                        System.IO.File.Delete(newOutputFilePath);   // 생성했던 newOutputFilePath 파일 삭제
                    }

                    acLyrTblRec.IsOff = true;   // 레이어 끄기

                    GC.Collect();   // 1회 반복 후 메모리 정리
                }

                // 레이어 상태를 복구함
                foreach (var layer in layerList)
                {
                    layer.Item1.IsOff = layer.Item2;
                }

                XtraMessageBox.Show("Plot을 완료했습니다.", "성공", MessageBoxButtons.OK, MessageBoxIcon.Information);
            }

            else
            {
                ed.WriteMessage("\n다른 플롯이 진행 중입니다.");
            }

            progressBarControl.Hide();
        }

        tr.Commit();
    }

    GC.Collect();
}

static bool IsWhiteImage(Mat image)
{
    // 이미지가 비어 있는지 확인
    if (image.Empty())
    {
        throw new ArgumentException("이미지를 읽을 수 없습니다.");
    }

    // 흰색 범위를 정의 (BGR)
    Scalar lowerWhite = new Scalar(255, 255, 255);
    Scalar upperWhite = new Scalar(255, 255, 255);

    // 흰색 범위 내의 픽셀들을 이진화
    Mat mask = new Mat();
    Cv2.InRange(image, lowerWhite, upperWhite, mask);

    // 흰색이 아닌 픽셀 수를 계산
    int nonWhitePixelCount = Cv2.CountNonZero(mask);

    // 모든 픽셀이 흰색인지 확인
    return nonWhitePixelCount == image.Total();
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
            button.BMPFileNameLgColor = @".\RibbonImage\DWGClipping_32x32.bmp";
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
            button.BMPFileNameLgColor = @".\RibbonImage\SendImagesToServer_32x32.bmp";
            button.Command = "SendToServer";
            button.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, button.GetType().Name, 1);

            row.Add(button);
            button.Dispose();

            // 구분자
            separator = new RibbonSeparator();
            separator.Style = CuiRibbonSeparatorStyle.LINE;
            separator.Uid = String.Format(CuiManager.Literals.uidCuiElementBase, separator.GetType().Name, 1);

            row.Add(separator);
            separator.Dispose();

            // 5번째 버튼
            button = new RibbonCommandItem();
            button.DefaultToolTip = "Receive Images from Client";
            button.DefaultHelpString = String.Format(CuiManager.Literals.defaultHelpBase, button.DefaultToolTip);
            button.Style = CuiRibbonButtonStyle.LARGE_WITH_TEXT;
            button.BMPFileNameLgColor = @".\RibbonImage\ReceiveFromClient_32x32.bmp";
            button.Command = "ReceiveFromClient";
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

* 소켓 전송 예제 (클라이언트)

![image](https://github.com/Soonbum/ObjectIRX_ManagedDotNetGuide/assets/16474083/7a021fca-4fac-4259-8155-812223d0224f)


```cs
public partial class ClientForm : DevExpress.XtraEditors.XtraForm
{
    bool bSendSuccess;

    public ClientForm()
    {
        InitializeComponent();
    }

    private void fileSelectButton_Click(object sender, EventArgs e)
    {
        // 파일 선택 버튼 클릭 시 파일 선택 다이얼로그를 띄움
        OpenFileDialog openFileDialog = new OpenFileDialog();
        openFileDialog.Title = "파일 선택";
        openFileDialog.Filter = "모든 파일(*.*)|*.*";
        openFileDialog.Multiselect = true;

        if (openFileDialog.ShowDialog() == DialogResult.OK)
        {
            // 선택한 모든 파일을 fileListBoxControl에 추가
            foreach (string fileName in openFileDialog.FileNames)
            {
                fileListBoxControl.Items.Add(fileName);
            }
        }
    }

    private void fileClearButton_Click(object sender, EventArgs e)
    {
        // 모두 삭제 버튼 클릭 시 fileListBoxControl의 모든 항목을 삭제
        fileListBoxControl.Items.Clear();
    }

    private async void sendButton_Click(object sender, EventArgs e)
    {
        bSendSuccess = false;
        progressBarControl.Position = 0;

        // 파일 보내기 버튼 클릭 시 서버IP/포트번호로 fileListBoxControl에 있는 파일들을 전송
        if (string.IsNullOrEmpty(textEdit_serverIP.Text) || string.IsNullOrEmpty(textEdit_serverPort.Text))
        {
            XtraMessageBox.Show("서버 IP 주소와 포트 번호를 입력해 주세요.", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
            return;
        }

        // 소켓을 열고 서버에 파일 전송
        // 프로젝트명 문자열을 먼저 보내고 fileListBoxControl에 있는 파일들을 전송
        try
        {
            try
            {
                using (TcpClient client = new TcpClient(textEdit_serverIP.Text, int.Parse(textEdit_serverPort.Text)))
                using (NetworkStream clientStream = client.GetStream())
                {
                    // Send project name
                    string projectName = textEdit_projectName.Text;
                    byte[] projectNameBytes = Encoding.UTF8.GetBytes(projectName);
                    byte[] projectNameLength = BitConverter.GetBytes(projectNameBytes.Length);
                    clientStream.Write(projectNameLength, 0, projectNameLength.Length); // 프로젝트 이름 길이 송신
                    clientStream.Write(projectNameBytes, 0, projectNameBytes.Length);   // 프로젝트 이름 송신

                    // 파일 개수 송신
                    int fileCount = fileListBoxControl.ItemCount;
                    byte[] fileCountBytes = BitConverter.GetBytes(fileCount);
                    clientStream.Write(fileCountBytes, 0, fileCountBytes.Length);

                    progressBarControl.Properties.Maximum = fileCount;

                    for (int i = 0; i < fileCount; i++)
                    {
                        string filePath = fileListBoxControl.Items[i].ToString();

                        // Send file name
                        byte[] fileNameBytes = Encoding.UTF8.GetBytes(filePath);
                        byte[] fileNameLength = BitConverter.GetBytes(fileNameBytes.Length);
                        clientStream.Write(fileNameLength, 0, fileNameLength.Length);   // 파일명 길이 송신
                        clientStream.Write(fileNameBytes, 0, fileNameBytes.Length);     // 파일명 송신

                        byte[] fileLength = BitConverter.GetBytes(new FileInfo(filePath).Length);
                        clientStream.Write(fileLength, 0, fileLength.Length);           // 파일 크기 송신

                        // Send file content
                        using (FileStream fs = new FileStream(filePath, FileMode.Open, FileAccess.Read))
                        {
                            byte[] buffer = new byte[1024];
                            int bytesRead;

                            while ((bytesRead = fs.Read(buffer, 0, buffer.Length)) > 0)
                            {
                                clientStream.Write(buffer, 0, bytesRead);  // 파일 내용 송신
                            }
                        }

                        await Task.Run((() => progressBarControl.PerformStep()));

                        GC.Collect();
                    }

                    bSendSuccess = true;
                }
            }
            catch (Exception ex)
            {
                // Handle exception
            }
        }
        catch (Exception ex)
        {
            XtraMessageBox.Show($"서버에 연결할 수 없습니다. ({ex.Message})", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
            return;
        }

        if (bSendSuccess)
            XtraMessageBox.Show("파일 전송이 완료되었습니다.", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
}
```

* SFTP 전송 예제 (클라이언트)

![image](https://github.com/Soonbum/ObjectIRX_ManagedDotNetGuide/assets/16474083/9a17bbb6-2ebc-484b-86c0-355727c6b0d4)


```cs
public partial class ClientForm : DevExpress.XtraEditors.XtraForm
{
    bool bSendSuccess;

    public ClientForm()
    {
        InitializeComponent();
    }

    private void fileSelectButton_Click(object sender, EventArgs e)
    {
        // 파일 선택 버튼 클릭 시 파일 선택 다이얼로그를 띄움
        OpenFileDialog openFileDialog = new OpenFileDialog();
        openFileDialog.Title = "파일 선택";
        openFileDialog.Filter = "모든 파일(*.*)|*.*";
        openFileDialog.Multiselect = true;

        if (openFileDialog.ShowDialog() == DialogResult.OK)
        {
            // 선택한 모든 파일을 fileListBoxControl에 추가
            foreach (string fileName in openFileDialog.FileNames)
            {
                fileListBoxControl.Items.Add(fileName);
            }
        }
    }

    private void fileClearButton_Click(object sender, EventArgs e)
    {
        // 모두 삭제 버튼 클릭 시 fileListBoxControl의 모든 항목을 삭제
        fileListBoxControl.Items.Clear();
    }

    private async void sendButton_Click(object sender, EventArgs e)
    {
        bSendSuccess = false;
        progressBarControl.Position = 0;

        // 파일 보내기 버튼 클릭 시 서버IP/포트번호로 fileListBoxControl에 있는 파일들을 전송
        if (string.IsNullOrEmpty(textEdit_serverIP.Text) || string.IsNullOrEmpty(textEdit_serverPort.Text))
        {
            XtraMessageBox.Show("서버 IP 주소와 포트 번호를 입력해 주세요.", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
            return;
        }

        // 소켓을 열고 서버에 파일 전송
        // 프로젝트명 문자열을 먼저 보내고 fileListBoxControl에 있는 파일들을 전송
        try
        {
            try
            {
                using (TcpClient client = new TcpClient(textEdit_serverIP.Text, int.Parse(textEdit_serverPort.Text)))
                using (NetworkStream clientStream = client.GetStream())
                {
                    // Send project name
                    string projectName = textEdit_projectName.Text;
                    byte[] projectNameBytes = Encoding.UTF8.GetBytes(projectName);
                    byte[] projectNameLength = BitConverter.GetBytes(projectNameBytes.Length);
                    clientStream.Write(projectNameLength, 0, projectNameLength.Length); // 프로젝트 이름 길이 송신
                    clientStream.Write(projectNameBytes, 0, projectNameBytes.Length);   // 프로젝트 이름 송신

                    // 파일 개수 송신
                    int fileCount = fileListBoxControl.ItemCount;
                    byte[] fileCountBytes = BitConverter.GetBytes(fileCount);
                    clientStream.Write(fileCountBytes, 0, fileCountBytes.Length);

                    progressBarControl.Properties.Maximum = fileCount;

                    // 리눅스 서버 SFTP 설정 방법은 다음과 같습니다.
                    // 1. sudo apt install vsftpd로 설치
                    // 2. ps aux | grep vsftpd로 확인
                    // 3. systemctl restart vsftpd
                    // 4. systemctl enable vsftpd

                    // 저장 경로: /home/계정명/프로젝트명
                    string remoteDirectory = "/home/" + textEdit_id.Text + "/" + textEdit_projectName.Text;

                    // 접속 정보
                    // 서버 IP: textEdit_serverIP.Text
                    // 서버 포트: textEdit_serverPort.Text
                    // 서버 ID: textEdit_id.Text
                    // 서버 Password: textEdit_password.Text
                    using (var sftp = new SftpClient(textEdit_serverIP.Text, int.Parse(textEdit_serverPort.Text), textEdit_id.Text, textEdit_password.Text))
                    {
                        try
                        {
                            sftp.Connect();

                            if (!sftp.Exists(remoteDirectory))
                            {
                                sftp.CreateDirectory(remoteDirectory);
                            }

                            for (int i = 0; i < fileCount; i++)
                            {
                                string filePath = fileListBoxControl.Items[i].ToString();

                                string fileName = Path.GetFileName(filePath);
                                string remoteFilePath = remoteDirectory.TrimEnd('/') + "/" + Path.GetFileName(fileName);

                                using (FileStream fs = new FileStream(filePath, FileMode.Open, FileAccess.Read))
                                {
                                    sftp.UploadFile(fs, remoteFilePath, true);
                                }

                                await Task.Run((() => progressBarControl.PerformStep()));

                                GC.Collect();
                            }

                            sftp.Disconnect();
                        }
                        catch (SftpPermissionDeniedException ep)
                        {
                            Console.WriteLine($"Permission Denied: {ep.Message}");
                        }
                        catch (Exception ex)
                        {
                            XtraMessageBox.Show($"SFTP 연결에 실패했습니다. ({ex.Message})", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
                            return;
                        }
                    }

                    bSendSuccess = true;
                }
            }
            catch (Exception ex)
            {
                // Handle exception
            }
        }
        catch (Exception ex)
        {
            XtraMessageBox.Show($"서버에 연결할 수 없습니다. ({ex.Message})", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
            return;
        }

        if (bSendSuccess)
            XtraMessageBox.Show("파일 전송이 완료되었습니다.", "알림", MessageBoxButtons.OK, MessageBoxIcon.Information);
    }
}
```

* 소켓 전송 예제 (서버)

![image](https://github.com/Soonbum/ObjectIRX_ManagedDotNetGuide/assets/16474083/4716bc32-9fa2-4e16-af0e-6980c3be6d6f)

```cs
public partial class ServerForm : DevExpress.XtraEditors.XtraForm
{
    private TcpListener listener;
    private Thread listenerThread;
    private bool isRunning;

    public ServerForm()
    {
        InitializeComponent();

        // 서버 IP 주소를 가져옴
        string serverIP = System.Net.Dns.GetHostAddresses(System.Net.Dns.GetHostName()).FirstOrDefault(ip => ip.AddressFamily == System.Net.Sockets.AddressFamily.InterNetwork).ToString();
        textEdit_serverIP.Text = serverIP;
    }

    private void button_selectFilePath_Click(object sender, EventArgs e)
    {
        // 저장 경로 지정 버튼 클릭시 저장 경로 지정 다이얼로그를 띄움
        FolderBrowserDialog folderBrowserDialog = new FolderBrowserDialog();
        folderBrowserDialog.Description = "저장 경로 지정";

        if (folderBrowserDialog.ShowDialog() == DialogResult.OK)
        {
            // 선택한 경로를 textEdit_savePath에 표시
            textEdit_savePath.Text = folderBrowserDialog.SelectedPath;
        }
    }

    private void checkButton_receiveFiles_CheckedChanged(object sender, EventArgs e)
    {
        // 저장 경로가 지정되어 있지 않거나 유효한 경로가 아니면 경고 메시지 출력하고 체크 해제
        if (string.IsNullOrEmpty(textEdit_savePath.Text) || !System.IO.Directory.Exists(textEdit_savePath.Text))
        {
            checkButton_receiveFiles.Checked = false;
            checkButton_receiveFiles.Text = "파일 수신 (중지)";
        }
        else
        {
            IPAddress ipAddr = IPAddress.Parse(textEdit_serverIP.Text);
            int port = Convert.ToInt32(textEdit_serverPort.Text);

            if (checkButton_receiveFiles.Checked)
            {
                checkButton_receiveFiles.Text = "파일 수신 (대기중)";
                button_selectFilePath.Enabled = false;

                if (!isRunning)
                {
                    listener = new TcpListener(ipAddr, port);
                    listener.Start();
                    isRunning = true;
                    listenerThread = new Thread(new ThreadStart(ListenForClients));
                    listenerThread.Start();
                }
            }
            else
            {
                checkButton_receiveFiles.Text = "파일 수신 (중지)";
                button_selectFilePath.Enabled = true;

                listener.Stop();
                isRunning = false;
            }
        }
    }

    private void ListenForClients()
    {
        while (isRunning)
        {
            try
            {
                var client = listener.AcceptTcpClient();
                var clientThread = new Thread(new ParameterizedThreadStart(HandleClientComm));
                clientThread.Start(client);
            }
            catch (Exception ex)
            {
                // Handle exception
            }
        }
    }

    private void HandleClientComm(object clientObj)
    {
        TcpClient client = (TcpClient)clientObj;
        NetworkStream clientStream = client.GetStream();
        byte[] buffer = new byte[1024];
        int bytesRead;
        long totalBytesReceived;

        // 프로젝트 이름 길이 수신
        bytesRead = clientStream.Read(buffer, 0, 4);
        int projectNameLength = BitConverter.ToInt32(buffer, 0);

        // 프로젝트 이름 수신
        bytesRead = clientStream.Read(buffer, 0, projectNameLength);
        string projectName = Encoding.UTF8.GetString(buffer, 0, bytesRead);

        // 저장 디렉토리 생성
        string projectPath = Path.Combine(textEdit_savePath.Text, projectName);
        Directory.CreateDirectory(projectPath);

        // 파일 개수 수신
        bytesRead = clientStream.Read(buffer, 0, 4);
        int fileCount = BitConverter.ToInt32(buffer, 0);

        // Read files
        for (int i = 0; i < fileCount; i++)
        {
            bytesRead = clientStream.Read(buffer, 0, 4);                        // 파일명 길이 수신
            if (bytesRead == 0) break;
            int fileNameLength = BitConverter.ToInt32(buffer, 0);
            bytesRead = clientStream.Read(buffer, 0, fileNameLength);           // 파일명 수신
            if (bytesRead == 0) break;
            string filePath = Encoding.UTF8.GetString(buffer, 0, bytesRead);

            string fileName = Path.GetFileName(filePath);
            string savePath = Path.Combine(projectPath, fileName);

            bytesRead = clientStream.Read(buffer, 0, 8);
            long fileLength = BitConverter.ToInt64(buffer, 0);                  // 파일 크기 수신

            using (FileStream fs = new FileStream(savePath, FileMode.Create, FileAccess.Write))
            {
                totalBytesReceived = 0;

                do
                {
                    if (totalBytesReceived <= fileLength - buffer.Length)
                        bytesRead = clientStream.Read(buffer, 0, buffer.Length);
                    else
                        bytesRead = clientStream.Read(buffer, 0, (int)(fileLength - totalBytesReceived));

                    totalBytesReceived += bytesRead;

                    if (bytesRead > 0)
                        fs.Write(buffer, 0, bytesRead);

                } while (bytesRead > 0);
            }

            GC.Collect();
        }

        client.Close();
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

* 영역 (Regions) 생성
  - ```cs
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

        // Create an in memory circle
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 2, 0);
            acCirc.Radius = 5;

            // Adds the circle to an object array
            DBObjectCollection acDBObjColl = new DBObjectCollection();
            acDBObjColl.Add(acCirc);

            // Calculate the regions based on each closed loop
            DBObjectCollection myRegionColl = new DBObjectCollection();
            myRegionColl = Region.CreateFromCurves(acDBObjColl);
            Region acRegion = myRegionColl[0] as Region;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acRegion);
            acTrans.AddNewlyCreatedDBObject(acRegion, true);

            // Dispose of the in memory circle not appended to the database
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - ```cs
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

        // Create two in memory circles
        using (Circle acCirc1 = new Circle())
        {
            acCirc1.Center = new Point3d(4, 4, 0);
            acCirc1.Radius = 2;

            using (Circle acCirc2 = new Circle())
            {
                acCirc2.Center = new Point3d(4, 4, 0);
                acCirc2.Radius = 1;

                // Adds the circle to an object array
                DBObjectCollection acDBObjColl = new DBObjectCollection();
                acDBObjColl.Add(acCirc1);
                acDBObjColl.Add(acCirc2);

                // Calculate the regions based on each closed loop
                DBObjectCollection myRegionColl = new DBObjectCollection();
                myRegionColl = Region.CreateFromCurves(acDBObjColl);
                Region acRegion1 = myRegionColl[0] as Region;
                Region acRegion2 = myRegionColl[1] as Region;

                // Subtract region 1 from region 2
                if (acRegion1.Area > acRegion2.Area)
                {
                    // Subtract the smaller region from the larger one
                    acRegion1.BooleanOperation(BooleanOperationType.BoolSubtract, acRegion2);
                    acRegion2.Dispose();

                    // Add the final region to the database
                    acBlkTblRec.AppendEntity(acRegion1);
                    acTrans.AddNewlyCreatedDBObject(acRegion1, true);
                }
                else
                {
                    // Subtract the smaller region from the larger one
                    acRegion2.BooleanOperation(BooleanOperationType.BoolSubtract, acRegion1);
                    acRegion1.Dispose();

                    // Add the final region to the database
                    acBlkTblRec.AppendEntity(acRegion2);
                    acTrans.AddNewlyCreatedDBObject(acRegion2, true);
                }

                // Dispose of the in memory objects not appended to the database
            }
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```

* 해치 (Hatches) 생성
  - ```cs
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

        // Create a circle object for the closed boundary to hatch
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(3, 3, 0);
            acCirc.Radius = 1;

            // Add the new circle object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);

            // Adds the circle to an object id array
            ObjectIdCollection acObjIdColl = new ObjectIdCollection();
            acObjIdColl.Add(acCirc.ObjectId);

            // Create the hatch object and append it to the block table record
            using (Hatch acHatch = new Hatch())
            {
                acBlkTblRec.AppendEntity(acHatch);
                acTrans.AddNewlyCreatedDBObject(acHatch, true);

                // Set the properties of the hatch object
                // Associative must be set after the hatch object is appended to the 
                // block table record and before AppendLoop
                acHatch.SetHatchPattern(HatchPatternType.PreDefined, "ANSI31");
                acHatch.Associative = true;
                acHatch.AppendLoop(HatchLoopTypes.Outermost, acObjIdColl);
                acHatch.EvaluateHatch(true);
            }
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```

* 선택(Selection)하기
  - PickFirst 선택 세트 가져오기
    ```cs
    // 주의사항
    // 1. PICKFIRST 시스템 변수가 1이어야 함
    // 2. 함수 위에 UsePickSet 커맨드 flag를 사용해야 함 (예: [CommandMethod("CheckForPickfirstSelection", CommandFlags.UsePickSet)])
    // 3. PickFirst 선택 세트를 가져오기 위해 SelectImplied 메서드를 사용해야 함
    
    // Get the current document
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Get the PickFirst selection set
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.SelectImplied();

    SelectionSet acSSet;

    // If the prompt status is OK, objects were selected before the command was started
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects in Pickfirst selection: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects in Pickfirst selection: 0");
    }

    // Clear the PickFirst selection set
    ObjectId[] idarrayEmpty = new ObjectId[0];
    acDocEd.SetImpliedSelection(idarrayEmpty);

    // Request for objects to be selected in the drawing area
    acSSPrompt = acDocEd.GetSelection();

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 도면 영역 내 오브젝트 선택하기
    ```cs
    // 선택하기 위한 함수는 다음과 같다.
    // - GetSelection: 화면에서 오브젝트를 집으라고 사용자에게 알려줌
    // - SelectAll: 도면 내 모든 오브젝트를 선택함
    // - SelectCrossingPolygon: 다각형 안에 있거나 가로지르는 오브젝트를 선택함
    // - SelectCrossingWindow: 사각형 영역 안에 있거나 가로지르는 오브젝트를 선택함
    // - SelectFence: 선택 펜스를 가로지르는 모든 오브젝트를 선택함
    // - SelectLast: 현재 공간에서 생성된 마지막 오브젝트를 선택함
    // - SelectPrevious: 이전 선택 동적에 의해 선택된 모든 오브젝트를 선택함
    // - SelectWindow: 사각형 영역 안에 있는 오브젝트를 선택함
    // - SelectWindowPolygon: 다각형 안에 있는 오브젝트를 선택함
    // - SelectAtPoint: 주어진 점을 통과하는 오브젝트를 선택함
    // - SelectByPolygon: 펜스 안쪽에 있는 오브젝트를 선택함
    
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Request for objects to be selected in the drawing area
        PromptSelectionResult acSSPrompt = acDoc.Editor.GetSelection();

        // If the prompt status is OK, objects were selected
        if (acSSPrompt.Status == PromptStatus.OK)
        {
            SelectionSet acSSet = acSSPrompt.Value;

            // Step through the objects in the selection set
            foreach (SelectedObject acSSObj in acSSet)
            {
                // Check to make sure a valid SelectedObject object was returned
                if (acSSObj != null)
                {
                    // Open the selected object for write
                    Entity acEnt = acTrans.GetObject(acSSObj.ObjectId, OpenMode.ForWrite) as Entity;

                    if (acEnt != null)
                    {
                        // Change the object's color to Green
                        acEnt.ColorIndex = 3;
                    }
                }
            }

            // Save the new object to the database
            acTrans.Commit();
        }

        // Dispose of the transaction
    }
    ```
  - 선택 세트 키워드
    ```cs
    private static void SelectionKeywordInputHandler(object sender, SelectionTextInputEventArgs eSelectionInput)
    {
        // Gets the current document editor and define other variables for the current scope
        Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;
        PromptSelectionResult acSSPrompt = null;
        SelectionSet acSSet = null;
        ObjectId[] acObjIds = null;

        // See if the user choose the myFence keyword
        switch (eSelectionInput.Input)
        {
            case "myFence":
                // Uses the four points to define a fence selection
                Point3dCollection ptsFence = new Point3dCollection();
                ptsFence.Add(new Point3d(5.0, 5.0, 0.0));
                ptsFence.Add(new Point3d(13.0, 15.0, 0.0));
                ptsFence.Add(new Point3d(12.0, 9.0, 0.0));
                ptsFence.Add(new Point3d(5.0, 5.0, 0.0));

                acSSPrompt = acDocEd.SelectFence(ptsFence);
                break;
            case "myWindow":
                // Defines a rectangular window selection
                acSSPrompt = acDocEd.SelectWindow(new Point3d(1.0, 1.0, 0.0), new Point3d(30.0, 20.0, 0.0));
                break;
            case "myWPoly":
                // Uses the four points to define a polygon window selection
                Point3dCollection ptsPolygon = new Point3dCollection();
                ptsPolygon.Add(new Point3d(5.0, 5.0, 0.0));
                ptsPolygon.Add(new Point3d(13.0, 15.0, 0.0));
                ptsPolygon.Add(new Point3d(12.0, 9.0, 0.0));
                ptsPolygon.Add(new Point3d(5.0, 5.0, 0.0));

                acSSPrompt = acDocEd.SelectWindowPolygon(ptsPolygon);
                break;
            case "myLastSel":
                // Gets the last object created
                acSSPrompt = acDocEd.SelectLast();
                break;
            case "myPrevSel":
                // Gets the previous object selection set
                acSSPrompt = acDocEd.SelectPrevious();
                break;
        }

        // If the prompt status is OK, objects were selected and return
        if (acSSPrompt != null)
        {
            if (acSSPrompt.Status == PromptStatus.OK)
            {
                // Objects were selected, so add them to the current selection
                acSSet = acSSPrompt.Value;
                acObjIds = acSSet.GetObjectIds();
                eSelectionInput.AddObjects(acObjIds);
            }
        }
    }

    [CommandMethod("SelectionKeywordInput")]
    public static void SelectionKeywordInput()
    {
        // Gets the current document editor
        Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

        // Setups the keyword options
        PromptSelectionOptions acKeywordOpts = new PromptSelectionOptions();
        acKeywordOpts.Keywords.Add("myFence");
        acKeywordOpts.Keywords.Add("myWindow");
        acKeywordOpts.Keywords.Add("myWPoly");
        acKeywordOpts.Keywords.Add("myLastSel");
        acKeywordOpts.Keywords.Add("myPrevSel");

        // Adds the event handler for keyword input
        acKeywordOpts.KeywordInput += new SelectionTextInputEventHandler(SelectionKeywordInputHandler);

        // Prompts the user for a selection set
        PromptSelectionResult acSSPrompt = acDocEd.GetSelection(acKeywordOpts);

        // If the prompt status is OK, objects were selected
        if (acSSPrompt.Status == PromptStatus.OK)
        {
            // Gets the selection set
            SelectionSet acSSet = acSSPrompt.Value;

            // Gets the objects from the selection set
            ObjectId[] acObjIds = acSSet.GetObjectIds();
            Database acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

            // Starts a transaction
            using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
            {
                try
                {
                    // Gets information about each object
                    foreach (ObjectId acObjId in acObjIds)
                    {
                        Entity acEnt = (Entity)acTrans.GetObject(acObjId, OpenMode.ForWrite, true);
                        acDocEd.WriteMessage("\nObject selected: " + acEnt.GetType().FullName);

                    }
                }
                finally
                {
                    acTrans.Dispose();
                }
            }
        }

        // Removes the event handler for keyword input
        acKeywordOpts.KeywordInput -= new SelectionTextInputEventHandler(SelectionKeywordInputHandler);
    }
    ```
  - 다중 선택 세트
    ```cs
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection();

    SelectionSet acSSet1;
    ObjectIdCollection acObjIdColl = new ObjectIdCollection();

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        // Get the selected objects
        acSSet1 = acSSPrompt.Value;

        // Append the selected objects to the ObjectIdCollection
        acObjIdColl = new ObjectIdCollection(acSSet1.GetObjectIds());
    }

    // Request for objects to be selected in the drawing area
    acSSPrompt = acDocEd.GetSelection();

    SelectionSet acSSet2;

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        acSSet2 = acSSPrompt.Value;

        // Check the size of the ObjectIdCollection, if zero, then initialize it
        if (acObjIdColl.Count == 0)
        {
            acObjIdColl = new ObjectIdCollection(acSSet2.GetObjectIds());
        }
        else
        {
            // Step through the second selection set
            foreach (ObjectId acObjId in acSSet2.GetObjectIds())
            {
                // Add each object id to the ObjectIdCollection
                acObjIdColl.Add(acObjId);
            }
        }
    }

    Application.ShowAlertDialog("Number of objects selected: " + acObjIdColl.Count.ToString());
    ```
  - 선택 필터
    ```cs
    // 선택 필터는 TypedValues의 인자의 조합으로 구성한다.
    // 1번째 인자: 필터 타입: object 등
      // 0 (DxfCode.Start): 오브젝트 타입을 의미함. "Line", "Circle", "Arc" 등
      // 2 (DxfCode.BlockName): 삽입 레퍼런스의 블록 이름을 의미함
      // 8 (DxfCode.LayerName): 레이어 이름을 의미함
      // 60 (DxfCode.Visibility): 오브젝트 가시성을 의미함 (0 = visible, 1 = invisible)
      // 62 (DxfCode.Color): 컬러값 0 ~ 256 중 하나. (0은 BYBLOCK, 256은 BYLAYER, 음수는 레이어가 꺼져 있음을 의미함)
      // 67: 모델/페이퍼 공간을 의미함 (0 또는 생략하면 model 공간, 1은 paper 공간)
    // 2번째 인자: 필터링할 값: circle 등
    
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[1];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "CIRCLE"), 0);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 다중 선택 필터
    ```cs
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[3];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Color, 5), 0);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "CIRCLE"), 1);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.LayerName, "0"), 2);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 선택 필터 - 비교 연산, 논리 연산을 적용한 예시 1
    ```cs
    // TypedValue의 인자의 조합을 이용한다.
      // 1번째 인자: DxfCode.Operator
      // 2번째 인자
      //   "*" (모두 허용), "=" (동일함), "!=", "/=", "<>" (동일하지 않음), "<" (미만), "<=" (이하), ">" (초과), ">=" (이상), "&" (Bitwise AND), "&=" (Bitwise masked equals)
    
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[3];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "CIRCLE"), 0);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Operator, ">="), 1);
    acTypValAr.SetValue(new TypedValue(40, 5), 2);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 선택 필터 - 비교 연산, 논리 연산을 적용한 예시 2
    ```cs
    // TypedValue의 인자의 조합을 이용한다.
      // 1번째 인자: DxfCode.Operator
      // 2번째 인자
      // 다음은 열고 닫는 연산자가 반드시 pair로 존재해야 한다.
      //   "<AND" 다수의 피연산자 "AND>"
      //   "<OR" 다수의 피연산자 "OR>"
      //   "<XOR" 2개의 피연산자 "XOR>"
      //   "<NOT" 1개의 피연산자 "NOT>"

    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[4];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Operator, "<or"), 0);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "TEXT"), 1);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "MTEXT"), 2);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Operator, "or>"), 3);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 선택 필터 - 와일드-카드 패턴 예시
    ```cs
    // # (pound): 하나의 숫자와 일치함
    // @ (at): 하나의 알파벳 문자와 일치함
    // . (period): 하나의 (알파벳도 아니고 숫자도 아닌) 문자와 일치함
    // * (asterisk): 모든 문자 시퀀스와 일치함 (빈 문자열도 포함됨)
    // ? (question mark): 하나의 문자와 일치함
    // ~ (tilde): 이것이 만약 패턴의 1번째 문자라면, 이것은 패턴을 제외한 모든 것과 일치함
    // [...]: 괄호 안에 포함된 문자들 중 하나와 일치함
    // [~...]: 괄호 안에 포함되지 않은 문자들 중 하나와 일치함
    // - (hyphen): 괄호 안에서 단일 문자에 대한 범위를 지정할 때 사용함
    // , (comma): 2개의 패턴을 분리함
    // ` (reverse quote): 특수 문자 이스케이프 용도
    
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[2];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "MTEXT"), 0);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Text, "*The*"), 1);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```
  - 선택 필터 - 외부 애플리케이션이 추가한 데이터 필터링하기
    ```cs
    // Get the current document editor
    Editor acDocEd = Application.DocumentManager.MdiActiveDocument.Editor;

    // Create a TypedValue array to define the filter criteria
    TypedValue[] acTypValAr = new TypedValue[2];
    acTypValAr.SetValue(new TypedValue((int)DxfCode.Start, "Circle"), 0);
    acTypValAr.SetValue(new TypedValue((int)DxfCode.ExtendedDataRegAppName, "MY_APP"), 1);

    // Assign the filter criteria to a SelectionFilter object
    SelectionFilter acSelFtr = new SelectionFilter(acTypValAr);

    // Request for objects to be selected in the drawing area
    PromptSelectionResult acSSPrompt;
    acSSPrompt = acDocEd.GetSelection(acSelFtr);

    // If the prompt status is OK, objects were selected
    if (acSSPrompt.Status == PromptStatus.OK)
    {
        SelectionSet acSSet = acSSPrompt.Value;

        Application.ShowAlertDialog("Number of objects selected: " + acSSet.Count.ToString());
    }
    else
    {
        Application.ShowAlertDialog("Number of objects selected: 0");
    }
    ```

* 오브젝트 편집하기
  - 비참조 오브젝트 제거하기
    ```cs
    // Purge 메서드는 ObjectIdCollection 또는 ObjectIdGraph 오브젝트 형태에서 여러 개의 오브젝트를 제거할 수 있음
    
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        // Create an ObjectIdCollection to hold the object ids for each table record
        ObjectIdCollection acObjIdColl = new ObjectIdCollection();

        // Step through each layer and add iterator to the ObjectIdCollection
        foreach (ObjectId acObjId in acLyrTbl)
        {
            acObjIdColl.Add(acObjId);
        }

        // Remove the layers that are in use and return the ones that can be erased
        acCurDb.Purge(acObjIdColl);

        // Step through the returned ObjectIdCollection
        // and erase each unreferenced layer
        foreach (ObjectId acObjId in acObjIdColl)
        {
            SymbolTableRecord acSymTblRec;
            acSymTblRec = acTrans.GetObject(acObjId, OpenMode.ForWrite) as SymbolTableRecord;

            try
            {
                // Erase the unreferenced layer
                acSymTblRec.Erase(true);
            }
            catch (Autodesk.AutoCAD.Runtime.Exception Ex)
            {
                // Layer could not be deleted
                Application.ShowAlertDialog("Error:\n" + Ex.Message);
            }
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 오브젝트 이름 변경하기
    ```cs
    // 오브젝트 이름은 최대 255 글자까지 가능함
    // 다음의 특수 문자는 오브젝트 이름에 사용할 수 없음: < > / \ " : ; ? , * | = ' 그리고 Unicode 글꼴로 생성된 특수 문자

    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Returns the layer table for the current database
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId,
                                        OpenMode.ForWrite) as LayerTable;

        // Clone layer 0 (copy it and its properties) as a new layer
        LayerTableRecord acLyrTblRec;
        acLyrTblRec = acTrans.GetObject(acLyrTbl["0"],
                                        OpenMode.ForRead).Clone() as LayerTableRecord;

        // Change the name of the cloned layer
        acLyrTblRec.Name = "MyLayer";

        // Add the cloned layer to the Layer table and transaction
        acLyrTbl.Add(acLyrTblRec);
        acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);

        // Save changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 오브젝트 삭제하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId,
                                        OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace],
                                        OpenMode.ForWrite) as BlockTableRecord;

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(2, 4), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(4, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(6, 4), 0, 0, 0);

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);

            // Update the display and display an alert message
            acDoc.Editor.Regen();
            Application.ShowAlertDialog("Erase the newly added polyline.");

            // Erase the polyline from the drawing
            acPoly.Erase(true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - 단일 오브젝트 복사하기
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

        // Create a circle that is at 2,3 with a radius of 4.25
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 3, 0);
            acCirc.Radius = 4.25;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);

            // Create a copy of the circle and change its radius
            Circle acCircClone = acCirc.Clone() as Circle;
            acCircClone.Radius = 1;

            // Add the cloned circle
            acBlkTblRec.AppendEntity(acCircClone);
            acTrans.AddNewlyCreatedDBObject(acCircClone, true);
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - 여러 오브젝트 복사하기
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

        // Create a circle that is at (0,0,0) with a radius of 5
        using (Circle acCirc1 = new Circle())
        {
            acCirc1.Center = new Point3d(0, 0, 0);
            acCirc1.Radius = 5;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc1);
            acTrans.AddNewlyCreatedDBObject(acCirc1, true);

            // Create a circle that is at (0,0,0) with a radius of 7
            using (Circle acCirc2 = new Circle())
            {
                acCirc2.Center = new Point3d(0, 0, 0);
                acCirc2.Radius = 7;

                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acCirc2);
                acTrans.AddNewlyCreatedDBObject(acCirc2, true);

                // Add all the objects to clone
                DBObjectCollection acDBObjColl = new DBObjectCollection();
                acDBObjColl.Add(acCirc1);
                acDBObjColl.Add(acCirc2);

                foreach (Entity acEnt in acDBObjColl)
                {
                    Entity acEntClone;
                    acEntClone = acEnt.Clone() as Entity;
                    acEntClone.ColorIndex = 1;

                    // Create a matrix and move each copied entity 15 units
                    acEntClone.TransformBy(Matrix3d.Displacement(new Vector3d(15, 0, 0)));

                    // Add the cloned object
                    acBlkTblRec.AppendEntity(acEntClone);
                    acTrans.AddNewlyCreatedDBObject(acEntClone, true);
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - 데이터베이스 간 오브젝트 복사하기
    ```cs
    // WblockCloneObjects 메서드는 다음 파라미터를 요구함
    //   ObjectIdCollection - 복사하고자 하는 오브젝트 리스트
    //   ObjectId - 복사하고자 하는 오브젝트들에 대한 새로운 부모 오브젝트의 ObjectId
    //   IdMapping - 복사할 오브젝트들에 대한 현재/새로운 ObjectId들의 데이터 구조
    //   DuplicateRecordCloning - 중복 오브젝트를 어떻게 처리해야 할지 결정함
    //   Defer Translation - object id 변환 여부를 제어함
    
    [CommandMethod("CopyObjectsBetweenDatabases", CommandFlags.Session)]
    public static void CopyObjectsBetweenDatabases()
    {
        ObjectIdCollection acObjIdColl = new ObjectIdCollection();

        // Get the current document and database
        Document acDoc = Application.DocumentManager.MdiActiveDocument;
        Database acCurDb = acDoc.Database;

        // Lock the current document
        using (DocumentLock acLckDocCur = acDoc.LockDocument())
        {
            // Start a transaction
            using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
            {
                // Open the Block table record for read
                BlockTable acBlkTbl;
                acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

                // Open the Block table record Model space for write
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

                // Create a circle that is at (0,0,0) with a radius of 5
                using (Circle acCirc1 = new Circle())
                {
                    acCirc1.Center = new Point3d(0, 0, 0);
                    acCirc1.Radius = 5;

                    // Add the new object to the block table record and the transaction
                    acBlkTblRec.AppendEntity(acCirc1);
                    acTrans.AddNewlyCreatedDBObject(acCirc1, true);

                    // Create a circle that is at (0,0,0) with a radius of 7
                    using (Circle acCirc2 = new Circle())
                    {
                        acCirc2.Center = new Point3d(0, 0, 0);
                        acCirc2.Radius = 7;

                        // Add the new object to the block table record and the transaction
                        acBlkTblRec.AppendEntity(acCirc2);
                        acTrans.AddNewlyCreatedDBObject(acCirc2, true);

                        // Add all the objects to copy to the new document
                        acObjIdColl = new ObjectIdCollection();
                        acObjIdColl.Add(acCirc1.ObjectId);
                        acObjIdColl.Add(acCirc2.ObjectId);
                    }
                }

                // Save the new objects to the database
                acTrans.Commit();
            }

            // Unlock the document
        }

        // Change the file and path to match a drawing template on your workstation
        string sLocalRoot = Application.GetSystemVariable("LOCALROOTPREFIX") as string;
        string sTemplatePath = sLocalRoot + "Template\\acad.dwt";

        // Create a new drawing to copy the objects to
        DocumentCollection acDocMgr = Application.DocumentManager;
        Document acNewDoc = acDocMgr.Add(sTemplatePath);
        Database acDbNewDoc = acNewDoc.Database;

        // Lock the new document
        using (DocumentLock acLckDoc = acNewDoc.LockDocument())
        {
            // Start a transaction in the new database
            using (Transaction acTrans = acDbNewDoc.TransactionManager.StartTransaction())
            {
                // Open the Block table for read
                BlockTable acBlkTblNewDoc;
                acBlkTblNewDoc = acTrans.GetObject(acDbNewDoc.BlockTableId, OpenMode.ForRead) as BlockTable;

                // Open the Block table record Model space for read
                BlockTableRecord acBlkTblRecNewDoc;
                acBlkTblRecNewDoc = acTrans.GetObject(acBlkTblNewDoc[BlockTableRecord.ModelSpace], OpenMode.ForRead) as BlockTableRecord;

                // Clone the objects to the new database
                IdMapping acIdMap = new IdMapping();
                acCurDb.WblockCloneObjects(acObjIdColl, acBlkTblRecNewDoc.ObjectId, acIdMap, DuplicateRecordCloning.Ignore, false);

                // Save the copied objects to the database
                acTrans.Commit();
            }

            // Unlock the document
        }

        // Set the new document current
        acDocMgr.MdiActiveDocument = acNewDoc;
    }
    ```
  - 오브젝트 오프셋 생성하기
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

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 2), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);
            acPoly.AddVertexAt(5, new Point2d(4, 1), 0, 0, 0);

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);

            // Offset the polyline a given distance
            DBObjectCollection acDbObjColl = acPoly.GetOffsetCurves(0.25);

            // Step through the new objects created
            foreach (Entity acEnt in acDbObjColl)
            {
                // Add each offset object
                acBlkTblRec.AppendEntity(acEnt);
                acTrans.AddNewlyCreatedDBObject(acEnt, true);
            }
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 행렬을 이용하여 오브젝트 변환하기
    * 4x4 변환 행렬인 Matrix3d 오브젝트와 TransformBy 메서드를 이용해 오브젝트를 이동, 회전, 미러링, 스케일링할 수 있다.
    * GetTransformedCopy 메서드를 이용해 엔티티 복사본을 만들고 복사본에게 변환 행렬을 적용할 수 있다.
    * 변환 행렬은 다음과 같다. (R = Rotation, T = Translation)
      R00 | R01 | R02 | T0
      R10 | R11 | R12 | T1
      R20 | R21 | R22 | T2
      0   | 0   | 0   | 1
    * 다음 변환 행렬은 점 (0, 0, 0)을 중심으로 90도 회전시키는 변환 행렬이다.
      0.0 | -1.0 | 0.0 | 0.0
      1.0 | 0.0  | 0.0 | 0.0
      0.0 | 0.0  | 1.0 | 0.0
      0.0 | 0.0  | 0.0 | 1.0
      - 이와 같은 변환 행렬을 만드는 방법은 2가지가 있다.
        ```cs
        double[] dMatrix = new double[16];

        dMatrix[0] = 0.0;
        dMatrix[1] = -1.0;
        dMatrix[2] = 0.0;
        dMatrix[3] = 0.0;
        dMatrix[4] = 1.0;
        dMatrix[5] = 0.0;
        dMatrix[6] = 0.0;
        dMatrix[7] = 0.0;
        dMatrix[8] = 0.0;
        dMatrix[9] = 0.0;
        dMatrix[10] = 1.0;
        dMatrix[11] = 0.0;
        dMatrix[12] = 0.0;
        dMatrix[13] = 0.0;
        dMatrix[14] = 0.0;
        dMatrix[15] = 1.0;
 
        Matrix3d acMat3d = new Matrix3d(dMatrix);
        ```
        ```cs
        Matrix3d acMat3d = new Matrix3d();
 
        acMat3d = Matrix3d.Rotation(Math.PI / 2, curUCS.Zaxis, new Point3d(0, 0, 0));
        ```
    * 다음은 변환 행렬의 몇 가지 예제이다.
      - Rotation Matrix: 45 degrees about point (5, 5, 0)
        0.707107 | -0.707107 | 0.0 | 5.0
        0.707107 | 0.707107  | 0.0 | -2.071068
        0.0      | 0.0       | 1.0 | 0.0
        0.0      | 0.0       | 0.0 | 1.0
      - Translation Matrix: move an entity by (10, 10, 0)
        1.0 | 0.0 | 0.0 | 10.0
        0.0 | 1.0 | 0.0 | 10.0
        0.0 | 0.0 | 1.0 | 0.0
        0.0 | 0.0 | 0.0 | 1.0
      - Scaling Matrix: scale by 10,10 at point (0, 0, 0)
        10.0 | 0.0  | 0.0  | 0.0
        0.0  | 10.0 | 0.0  | 0.0
        0.0  | 0.0  | 10.0 | 0.0
        0.0  | 0.0  | 0.0  | 1.0
      - Scaling Matrix: scale by 10,10 at point (2, 2, 0)
        10.0 | 0.0  | 0.0  | -18.0
        0.0  | 10.0 | 0.0  | -18.0
        0.0  | 0.0  | 10.0 | 0.0
        0.0  | 0.0  | 0.0  | 1.0
  - 오브젝트 이동하기
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

        // Create a circle that is at 2,2 with a radius of 0.5
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 2, 0);
            acCirc.Radius = 0.5;

            // Create a matrix and move the circle using a vector from (0,0,0) to (2,0,0)
            Point3d acPt3d = new Point3d(0, 0, 0);
            Vector3d acVec3d = acPt3d.GetVectorTo(new Point3d(2, 0, 0));

            acCirc.TransformBy(Matrix3d.Displacement(acVec3d));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 오브젝트 회전하기
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

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 3), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 3), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 3), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);
            acPoly.AddVertexAt(5, new Point2d(4, 2), 0, 0, 0);

            // Close the polyline
            acPoly.Closed = true;

            Matrix3d curUCSMatrix = acDoc.Editor.CurrentUserCoordinateSystem;
            CoordinateSystem3d curUCS = curUCSMatrix.CoordinateSystem3d;

            // Rotate the polyline 45 degrees, around the Z-axis of the current UCS
            // using a base point of (4,4.25,0)
            acPoly.TransformBy(Matrix3d.Rotation(0.7854,
                                                 curUCS.Zaxis,
                                                 new Point3d(4, 4.25, 0)));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 오브젝트 미러링
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

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 2), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);
            acPoly.AddVertexAt(5, new Point2d(4, 1), 0, 0, 0);

            // Create a bulge of -2 at vertex 1
            acPoly.SetBulgeAt(1, -2);

            // Close the polyline
            acPoly.Closed = true;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);

            // Create a copy of the original polyline
            Polyline acPolyMirCopy = acPoly.Clone() as Polyline;
            acPolyMirCopy.ColorIndex = 5;

            // Define the mirror line
            Point3d acPtFrom = new Point3d(0, 4.25, 0);
            Point3d acPtTo = new Point3d(4, 4.25, 0);
            Line3d acLine3d = new Line3d(acPtFrom, acPtTo);

            // Mirror the polyline across the X axis
            acPolyMirCopy.TransformBy(Matrix3d.Mirroring(acLine3d));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPolyMirCopy);
            acTrans.AddNewlyCreatedDBObject(acPolyMirCopy, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 오브젝트 스케일링
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

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 3), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 3), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 3), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);
            acPoly.AddVertexAt(5, new Point2d(4, 2), 0, 0, 0);

            // Close the polyline
            acPoly.Closed = true;

            // Reduce the object by a factor of 0.5 
            // using a base point of (4,4.25,0)
            acPoly.TransformBy(Matrix3d.Scaling(0.5, new Point3d(4, 4.25, 0)));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 오브젝트 연장 (Extend) 및 자르기 (Trim)
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

        // Create a line that starts at (4,4,0) and ends at (7,7,0)
        using (Line acLine = new Line(new Point3d(4, 4, 0),
                                new Point3d(7, 7, 0)))
        {

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acLine);
            acTrans.AddNewlyCreatedDBObject(acLine, true);

            // Update the display and diaplay a message box
            acDoc.Editor.Regen();
            Application.ShowAlertDialog("Before extend");

            // Double the length of the line
            acLine.EndPoint = acLine.EndPoint + acLine.Delta;
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - 오브젝트 분해하기 (Explode)
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

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 2), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);
            acPoly.AddVertexAt(5, new Point2d(4, 1), 0, 0, 0);

            // Sets the bulge at index 3
            acPoly.SetBulgeAt(3, -0.5);

            // Explodes the polyline
            DBObjectCollection acDBObjColl = new DBObjectCollection();
            acPoly.Explode(acDBObjColl);

            foreach (Entity acEnt in acDBObjColl)
            {
                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acEnt);
                acTrans.AddNewlyCreatedDBObject(acEnt, true);
            }

            // Dispose of the in memory polyline
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 폴리라인 편집하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId,
                                        OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a lightweight polyline
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 2), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(4, 4), 0, 0, 0);

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);

            // Sets the bulge at index 3
            acPoly.SetBulgeAt(3, -0.5);

            // Add a new vertex
            acPoly.AddVertexAt(5, new Point2d(4, 1), 0, 0, 0);

            // Sets the start and end width at index 4
            acPoly.SetStartWidthAt(4, 0.1);
            acPoly.SetEndWidthAt(4, 0.5);

            // Close the polyline
            acPoly.Closed = true;
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```

* 프로퍼티 (레이어) 조작하기
  - 레이어 이름 나열하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerNames = "";

        foreach (ObjectId acObjId in acLyrTbl)
        {
            LayerTableRecord acLyrTblRec;
            acLyrTblRec = acTrans.GetObject(acObjId, OpenMode.ForRead) as LayerTableRecord;

            sLayerNames = sLayerNames + "\n" + acLyrTblRec.Name;
        }

        Application.ShowAlertDialog("The layers in this drawing are: " + sLayerNames);

        // Dispose of the transaction
    }
    ```
  - 레이어 생성 및 이름 붙이기
    ```cs
    // 레이어 이름: 최대 255글자이며 글자, 숫자 및 일부 특수문자($, -, _)로 구성할 수 있음
    
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "Center";

        if (acLyrTbl.Has(sLayerName) == false)
        {
            using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
            {
                // Assign the layer the ACI color 3 and a name
                acLyrTblRec.Color = Color.FromColorIndex(ColorMethod.ByAci, 3);
                acLyrTblRec.Name = sLayerName;

                // Upgrade the Layer table for write
                acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                // Append the new layer to the Layer table and the transaction
                acLyrTbl.Add(acLyrTblRec);
                acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
            }
        }

        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a circle object
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 2, 0);
            acCirc.Radius = 1;
            acCirc.Layer = sLayerName;

            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 현재 레이어로 설정하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "Center";

        if (acLyrTbl.Has(sLayerName) == true)
        {
            // Set the layer Center current
            acCurDb.Clayer = acLyrTbl[sLayerName];

            // Save the changes
            acTrans.Commit();
        }

        // Dispose of the transaction
    }
    ```
  - 레이어 켜고 끄기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "ABC";

        if (acLyrTbl.Has(sLayerName) == false)
        {
            using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
            {
                // Assign the layer a name
                acLyrTblRec.Name = sLayerName;

                // Turn the layer off
                acLyrTblRec.IsOff = true;

                // Upgrade the Layer table for write
                acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                // Append the new layer to the Layer table and the transaction
                acLyrTbl.Add(acLyrTblRec);
                acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
            }
        }
        else
        {
            LayerTableRecord acLyrTblRec = acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForWrite) as LayerTableRecord;

            // Turn the layer off
            acLyrTblRec.IsOff = true;
        }

        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a circle object
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 2, 0);
            acCirc.Radius = 1;
            acCirc.Layer = sLayerName;

            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 레이어 동결/해동
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "ABC";

        if (acLyrTbl.Has(sLayerName) == false)
        {
            using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
            {
                // Assign the layer a name
                acLyrTblRec.Name = sLayerName;

                // Freeze the layer
                acLyrTblRec.IsFrozen = true;

                // Upgrade the Layer table for write
                acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                // Append the new layer to the Layer table and the transaction
                acLyrTbl.Add(acLyrTblRec);
                acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
            }
        }
        else
        {
            LayerTableRecord acLyrTblRec = acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForWrite) as LayerTableRecord;

            // Freeze the layer
            acLyrTblRec.IsFrozen = true;
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 레이어 잠금/잠금해제
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "ABC";

        if (acLyrTbl.Has(sLayerName) == false)
        {
            using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
            {
                // Assign the layer a name
                acLyrTblRec.Name = sLayerName;

                // Lock the layer
                acLyrTblRec.IsLocked = true;

                // Upgrade the Layer table for write
                acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                // Append the new layer to the Layer table and the transaction
                acLyrTbl.Add(acLyrTblRec);
                acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
            }
        }
        else
        {
            LayerTableRecord acLyrTblRec = acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForWrite) as LayerTableRecord;

            // Lock the layer
            acLyrTblRec.IsLocked = true;
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 레이어에 컬러 할당하기
    ```cs
    // ACI 컬러값: 0 ~ 256 중 하나. (0은 BYBLOCK, 256은 BYLAYER, 음수는 레이어가 꺼져 있음을 의미함)
    // ColorIndex 프로퍼티는 ACI 컬러값만 지원하지만, Color 프로퍼티는 컬러북 컬러, RGB 값도 지원함

    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        // Define an array of layer names
        string[] sLayerNames = new string[3];
        sLayerNames[0] = "ACIRed";
        sLayerNames[1] = "TrueBlue";
        sLayerNames[2] = "ColorBookYellow";

        // Define an array of colors for the layers
        Color[] acColors = new Color[3];
        acColors[0] = Color.FromColorIndex(ColorMethod.ByAci, 1);
        acColors[1] = Color.FromRgb(23, 54, 232);
        acColors[2] = Color.FromNames("PANTONE Yellow 0131 C",
                                      "PANTONE+ Pastels & Neons Coated");

        int nCnt = 0;

        // Add or change each layer in the drawing
        foreach (string sLayerName in sLayerNames)
        {
            if (acLyrTbl.Has(sLayerName) == false)
            {
                using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
                {
                    // Assign the layer a name
                    acLyrTblRec.Name = sLayerName;

                    // Set the color of the layer
                    acLyrTblRec.Color = acColors[nCnt];

                    // Upgrade the Layer table for write
                    if (acLyrTbl.IsWriteEnabled == false) acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                    // Append the new layer to the Layer table and the transaction
                    acLyrTbl.Add(acLyrTblRec);
                    acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
                }
            }
            else
            {
                // Open the layer if it already exists for write
                LayerTableRecord acLyrTblRec = acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForWrite) as LayerTableRecord;

                // Set the color of the layer
                acLyrTblRec.Color = acColors[nCnt];
            }

            nCnt = nCnt + 1;
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 레이어에 라인타입 할당하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "ABC";

        if (acLyrTbl.Has(sLayerName) == false)
        {
            using (LayerTableRecord acLyrTblRec = new LayerTableRecord())
            {
                // Assign the layer a name
                acLyrTblRec.Name = sLayerName;

                // Open the Layer table for read
                LinetypeTable acLinTbl;
                acLinTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

                if (acLinTbl.Has("Center") == true)
                {
                    // Set the linetype for the layer
                    acLyrTblRec.LinetypeObjectId = acLinTbl["Center"];
                }

                // Upgrade the Layer table for write
                acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForWrite);

                // Append the new layer to the Layer table and the transaction
                acLyrTbl.Add(acLyrTblRec);
                acTrans.AddNewlyCreatedDBObject(acLyrTblRec, true);
            }
        }
        else
        {
            LayerTableRecord acLyrTblRec = acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForRead) as LayerTableRecord;

            // Open the Layer table for read
            LinetypeTable acLinTbl;
            acLinTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

            if (acLinTbl.Has("Center") == true)
            {
                // Upgrade the Layer Table Record for write
                acTrans.GetObject(acLyrTbl[sLayerName], OpenMode.ForWrite);

                // Set the linetype for the layer
                acLyrTblRec.LinetypeObjectId = acLinTbl["Center"];
            }
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 레이어 삭제
    ```cs
    // 현재 레이어, 레이어 0, xref-의존 레이어, 오브젝트를 포함하는 레이어는 삭제할 수 없음
    // 레이어 삭제 후에 확실하게 삭제되었는지 확인하기 위해 Purge 함수를 사용하는 것을 권장함
    // 주의: 특수 레이어 DEFPOINTS와 함께 블록 정의가 참조하는 레이어는 보이는 레이어가 없다 하더라도 삭제할 수 없음
    
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Layer table for read
        LayerTable acLyrTbl;
        acLyrTbl = acTrans.GetObject(acCurDb.LayerTableId, OpenMode.ForRead) as LayerTable;

        string sLayerName = "ABC";

        if (acLyrTbl.Has(sLayerName) == true)
        {
            // Check to see if it is safe to erase layer
            ObjectIdCollection acObjIdColl = new ObjectIdCollection();
            acObjIdColl.Add(acLyrTbl[sLayerName]);
            acCurDb.Purge(acObjIdColl);

            if (acObjIdColl.Count > 0)
            {
                LayerTableRecord acLyrTblRec;
                acLyrTblRec = acTrans.GetObject(acObjIdColl[0], OpenMode.ForWrite) as LayerTableRecord;

                try
                {
                    // Erase the unreferenced layer
                    acLyrTblRec.Erase(true);

                    // Save the changes and dispose of the transaction
                    acTrans.Commit();
                }
                catch (Autodesk.AutoCAD.Runtime.Exception Ex)
                {
                    // Layer could not be deleted
                    Application.ShowAlertDialog("Error:\n" + Ex.Message);
                }
            }
        }
    }
    ```

* 프로퍼티 (컬러) 조작하기
  - 오브젝트에 컬러 값 할당하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Define an array of colors for the layers
        Color[] acColors = new Color[3];
        acColors[0] = Color.FromColorIndex(ColorMethod.ByAci, 1);
        acColors[1] = Color.FromRgb(23, 54, 232);
        acColors[2] = Color.FromNames("PANTONE Yellow 0131 C",
                                      "PANTONE(R) pastel coated");

        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a circle object and assign it the ACI value of 4
        Point3d acPt = new Point3d(0, 3, 0);
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = acPt;
            acCirc.Radius = 1;
            acCirc.ColorIndex = 4;

            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);

            int nCnt = 0;

            while (nCnt < 3)
            {
                // Create a copy of the circle
                Circle acCircCopy;
                acCircCopy = acCirc.Clone() as Circle;

                // Shift the copy along the Y-axis
                acPt = new Point3d(acPt.X, acPt.Y + 3, acPt.Z);
                acCircCopy.Center = acPt;

                // Assign the new color to the circle
                acCircCopy.Color = acColors[nCnt];

                acBlkTblRec.AppendEntity(acCircCopy);
                acTrans.AddNewlyCreatedDBObject(acCircCopy, true);

                nCnt = nCnt + 1;
            }
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 데이터베이스를 통해 현재 컬러로 설정하기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
    // Set the current color
    acDoc.Database.Cecolor = Color.FromColorIndex(ColorMethod.ByLayer, 256);
    ```
  - CECOLOR 시스템 변수를 통해 현재 컬러로 설정하기
    ```cs
    Application.SetSystemVariable("CECOLOR", "1");
    ```

* 프로퍼티 (라인타입) 조작하기
  - 라인타입 불러오기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Linetype table for read
        LinetypeTable acLineTypTbl;
        acLineTypTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

        string sLineTypName = "Center";

        if (acLineTypTbl.Has(sLineTypName) == false)
        {
            // Load the Center Linetype
            acCurDb.LoadLineTypeFile(sLineTypName, "acad.lin");
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 오브젝트에 라인타입 할당하기
    ```cs
    // 참고로 라인타입은 3가지가 있음: BYBLOCK, BYLAYER, CONTINUOUS

    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Linetype table for read
        LinetypeTable acLineTypTbl;
        acLineTypTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

        string sLineTypName = "Center";

        if (acLineTypTbl.Has(sLineTypName) == false)
        {
            acCurDb.LoadLineTypeFile(sLineTypName, "acad.lin");
        }

        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a circle object
        using (Circle acCirc = new Circle())
        {
            acCirc.Center = new Point3d(2, 2, 0);
            acCirc.Radius = 1;
            acCirc.Linetype = sLineTypName;

            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 데이터베이스를 통해 현재 라인타입으로 설정하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Linetype table for read
        LinetypeTable acLineTypTbl;
        acLineTypTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

        string sLineTypName = "Center";

        if (acLineTypTbl.Has(sLineTypName) == true)
        {
            // Set the linetype Center current
            acCurDb.Celtype = acLineTypTbl[sLineTypName];

            // Save the changes
            acTrans.Commit();
        }

        // Dispose of the transaction
    }
    ```
  - CELTYPE 시스템 변수를 통해 현재 라인타입으로 설정하기
    ```cs
    Application.SetSystemVariable("CELTYPE", "Center");
    ```
  - 라인타입 설명 변경하기
    ```cs
    // 라인타입 설명은 최대 47 글자 가능. 설명은 주석 또는 일련의 밑줄, 점, 대시 및 공백을 사용하여 선 유형 패턴을 간단하게 나타낼 수 있다.
    
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;
 
    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Linetype table record of the current linetype for write
        LinetypeTableRecord acLineTypTblRec;
        acLineTypTblRec = acTrans.GetObject(acCurDb.Celtype, OpenMode.ForWrite) as LinetypeTableRecord;
 
        // Change the description of the current linetype
        acLineTypTblRec.AsciiDescription = "Exterior Wall";
 
        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 라인타입 스케일 지정하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Save the current linetype
        ObjectId acObjId = acCurDb.Celtype;

        // Set the global linetype scale
        acCurDb.Ltscale = 3;

        // Open the Linetype table for read
        LinetypeTable acLineTypTbl;
        acLineTypTbl = acTrans.GetObject(acCurDb.LinetypeTableId, OpenMode.ForRead) as LinetypeTable;

        string sLineTypName = "Border";

        if (acLineTypTbl.Has(sLineTypName) == false)
        {
            acCurDb.LoadLineTypeFile(sLineTypName, "acad.lin");
        }

        // Set the Border linetype current
        acCurDb.Celtype = acLineTypTbl[sLineTypName];

        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a circle object and set its linetype
        // scale to half of full size
        using (Circle acCirc1 = new Circle())
        {
            acCirc1.Center = new Point3d(2, 2, 0);
            acCirc1.Radius = 4;
            acCirc1.LinetypeScale = 0.5;

            acBlkTblRec.AppendEntity(acCirc1);
            acTrans.AddNewlyCreatedDBObject(acCirc1, true);

            // Create a second circle object
            using (Circle acCirc2 = new Circle())
            {
                acCirc2.Center = new Point3d(12, 2, 0);
                acCirc2.Radius = 4;

                acBlkTblRec.AppendEntity(acCirc2);
                acTrans.AddNewlyCreatedDBObject(acCirc2, true);
            }
        }

        // Restore the original active linetype
        acCurDb.Celtype = acObjId;

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```

* 레이어 상태 저장/복구
  - 도면에 저장된 레이어 상태 리스팅
    ```cs
    ```
  - 레이어 상태 관리하기
    ```cs
    ```
  - 레이어 상태 저장하기
    ```cs
    ```
  - 레이어 상태 이름 변경하기
    ```cs
    ```
  - 레이어 상태 제거하기
    ```cs
    ```
  - 레이어 상태 복구하기
    ```cs
    ```
  - 저장된 레이어 상태 내보내기/가져오기
    ```cs
    ```

* 도면에 텍스트 추가하기
  - 멀티라인 텍스트 추가하기
    ```cs
    ```
  - 멀티라인 텍스트 포맷
    ```cs
    ```
  - 싱글라인 텍스트 생성하기
    ```cs
    ```
  - 싱글라인 텍스트 변경하기
    ```cs
    ```
  - 텍스트 높이 설정하기
    ```cs
    ```
  - 기울임 각도 설정하기
    ```cs
    ```
  - 텍스트 정렬하기
    ```cs
    ```
  - 텍스트 생성 플래그 설정하기
    ```cs
    ```
  - 텍스트 스타일 생성/변경하기
    ```cs
    ```
  - 글꼴 할당하기
    ```cs
    ```
  - TrueType 글꼴 사용하기
    ```cs
    ```
  - Unicode 및 큰 글꼴 사용하기
    ```cs
    ```
  - 대체용 글꼴
    ```cs
    ```
  - Unicode 문자, 제어 코드, 특수 문자 사용하기
    ```cs
    ```
  - 스펠링 체크
    ```cs
    ```

* 치수 및 공차

* 3D 공간
