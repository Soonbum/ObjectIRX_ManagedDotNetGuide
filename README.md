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

#### 레이어 관리
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

#### 도큐먼트 창 업데이트

```cs
// Redraw the drawing
Application.UpdateScreen();
Application.DocumentManager.MdiActiveDocument.Editor.UpdateScreen();
 
// Regenerate the drawing
Application.DocumentManager.MdiActiveDocument.Editor.Regen();
```

#### 사각형 영역 마우스로 입력 받고 이미지 파일로 플롯(인쇄)하기

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

#### 레이어 별로 이미지 Clipping해서 플롯(인쇄)하기

```cs
public ImageClippingForm()
{
    InitializeComponent();

    popupContainerEdit_PaperOrientation.Text = "가로";
    popupContainerEdit_PaperSize.Text = "ISO A3 (1123.00 x 1587.00 Pixels)";

    progressBarControl.Hide();
}

private async void ImageSaveButton_Click(object sender, EventArgs e)
{
    Document doc = IntelliCAD.ApplicationServices.Application.DocumentManager.MdiActiveDocument;
    Editor ed = doc.Editor;
    Database db = doc.Database;

    // 사용자로부터 두 점을 입력받아 화면 범위를 선택
    PromptPointResult ppr1 = ed.GetPoint("\n첫 번째 지점을 선택하십시오 ");

    if (ppr1.Status != PromptStatus.OK) return;
    Teigha.Geometry.Point3d firstPoint = ppr1.Value;

    if (ppr1.Status == PromptStatus.Cancel) return;

    PromptPointResult ppr2 = ed.GetCorner("\n두 번째 지점을 선택하십시오 ", ppr1.Value);

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
        int nWidthPixel = 0;
        int nHeightPixel = 0;
        bool isPaper = true;

        // 플롯 크기를 구함
        getPlotSize(popupContainerEdit_PaperSize.Text, out nWidthPixel, out nHeightPixel, out isPaper);

        // 용지 방향
        bool OrientationLandScape = (popupContainerEdit_PaperOrientation.Text == "가로") ? true : false;

        // 용지 크기
        string paperSize = popupContainerEdit_PaperSize.Text;

        // 파일 저장 다이얼로그 표시
        System.Windows.Forms.SaveFileDialog saveFileDialog = new System.Windows.Forms.SaveFileDialog()
        {
            Filter = "이미지 파일|*.png",
            Title = "이미지 파일을 저장하십시오.",
            FileName = "TestPrint.png"
        };

        if (saveFileDialog.ShowDialog() != DialogResult.OK)
            return;

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
        plotSettingsValidator.SetUseStandardScale(plotSettings, false);
        plotSettingsValidator.SetCustomPrintScale(plotSettings, new CustomScale(1.0, 1.0));
        plotSettings.PrintLineweights = true;
        plotSettingsValidator.SetStdScaleType(plotSettings, StdScaleType.ScaleToFit);
        plotSettingsValidator.SetPlotCentered(plotSettings, true);

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
            PerformPlot(outputFilePath, plotInfo, "TestPrint");

            // outputFilePath 이미지 파일을 그레이스케일로 변환
            if (checkEdit_grayscale.Checked)
            {
                colorImgToGrayscaleImg(outputFilePath);
            }

            // outputFilePath 이미지 파일에 안티앨리어싱 적용
            if (checkEdit_antialiasing.Checked)
            {
                imgAntialiasing(outputFilePath);
            }

            // 용지 크기가 아닌 사용자가 입력한 비율대로 정확하게 crop하기
            cropImage(outputFilePath, nWidthPixel, nHeightPixel, plotWindow, isPaper);

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
                PerformPlot(newOutputFilePath, plotInfo, "TestPrint");

                // newOutputFilePath 이미지 파일을 그레이스케일로 변환
                if (checkEdit_grayscale.Checked)
                {
                    colorImgToGrayscaleImg(newOutputFilePath);
                }

                // outputFilePath 이미지 파일에 안티앨리어싱 적용
                if (checkEdit_antialiasing.Checked)
                {
                    imgAntialiasing(newOutputFilePath);
                }

                // 흰색 이미지인 경우 파일 삭제
                Mat image = Cv2.ImRead(newOutputFilePath, ImreadModes.Color);
                if (IsWhiteImage(image))
                {
                    System.IO.File.Delete(newOutputFilePath);   // 생성했던 newOutputFilePath 파일 삭제
                }
                else
                {
                    // 용지 크기가 아닌 사용자가 입력한 비율대로 정확하게 crop하기
                    cropImage(newOutputFilePath, nWidthPixel, nHeightPixel, plotWindow, isPaper);
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
            ed.WriteMessage("\n다른 플롯이 진행 중입니다.\n");
        }

        progressBarControl.Hide();

        tr.Commit();
    }

    GC.Collect();

    ed.WriteMessage("\n");
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

void getPlotSize(string paperSizeName, out int plotWidth, out int plotHeight, out bool isPaper)
{
    plotWidth = 0; plotHeight = 0; isPaper = true;

    if (paperSizeName == "ISO A0 (3179.00 x 4494.00 Pixels)")
    {
        plotWidth = 3179; plotHeight = 4494; isPaper = true;
    }
    else if (paperSizeName == "ISO A1 (2245.00 x 3179.00 Pixels)")
    {
        plotWidth = 2245; plotHeight = 3179; isPaper = true;
    }
    else if (paperSizeName == "ISO A2 (1587.00 x 2245.00 Pixels)")
    {
        plotWidth = 1587; plotHeight = 2245; isPaper = true;
    }
    else if (paperSizeName == "ISO A3 (1123.00 x 1587.00 Pixels)")
    {
        plotWidth = 1123; plotHeight = 1587; isPaper = true;
    }
    else if (paperSizeName == "ISO A4 (794.00 x 1123.00 Pixels)")
    {
        plotWidth = 794; plotHeight = 1123; isPaper = true;
    }
    else if (paperSizeName == "FHD (1920.00 x 1080.00 Pixels)")
    {
        plotWidth = 1920; plotHeight = 1080; isPaper = false;
    }
    else if (paperSizeName == "WUXGA (1920.00 x 1200.00 Pixels)")
    {
        plotWidth = 1920; plotHeight = 1200; isPaper = false;
    }
    else if (paperSizeName == "4K UHD (3840.00 x 2160.00 Pixels)")
    {
        plotWidth = 3840; plotHeight = 2160; isPaper = false;
    }
    else if (paperSizeName == "4K Digital Cinema (4096.00 x 2160.00 Pixels)")
    {
        plotWidth = 4096; plotHeight = 2160; isPaper = false;
    }
    else if (paperSizeName == "8K UHDTV (7680.00 x 4320.00 Pixels)")
    {
        plotWidth = 7680; plotHeight = 4320; isPaper = false;
    }
}

void cropImage(string outputFilePath, int plotWidth, int plotHeight, Extents2d plotWindow, bool isPaper)
{
    // 실제 사용자가 그린 RECT 너비와 높이
    int rectWidth = (int)plotWindow.MaxPoint.X - (int)plotWindow.MinPoint.X;
    int rectHeight = (int)plotWindow.MaxPoint.Y - (int)plotWindow.MinPoint.Y;

    int cropLength = 0;

    if (isPaper)
    {
        // 용지 규격, 세로 방향
        // 용지 규격, 가로 방향 --> 이미지 세로 방향으로 나옴

        // plotHeight에서 위아래쪽을 crop해야 함
        // plotWidth : plotHeight - 2*(크롭길이) = rectWidth : rectHeight
        cropLength = (rectWidth * plotHeight - plotWidth * rectHeight) / (2 * rectWidth);

        Rect topBottomCropRegion = new Rect(0, cropLength, plotWidth, plotHeight - cropLength * 2);
        Mat afterProcessImage = Cv2.ImRead(outputFilePath);

        if (IsValidCropRegion(afterProcessImage, topBottomCropRegion))
        {
            Mat topBottomCroppedImage = new Mat(afterProcessImage, topBottomCropRegion);
            Cv2.ImWrite(outputFilePath, topBottomCroppedImage);
        }
    }
    else
    {
        // 화면 규격, 가로 방향
        // 화면 규격, 세로 방향 --> 이미지 가로 방향으로 나옴

        // plotWidth에서 양쪽을 crop해야 함
        // plotWidth - 2*(크롭길이) : plotHeight = rectWidth : rectHeight
        cropLength = (plotWidth - (rectWidth * plotHeight / rectHeight)) / 2;

        Rect leftRightCropRegion = new Rect(cropLength, 0, plotWidth - cropLength * 2, plotHeight);
        Mat afterProcessImage = Cv2.ImRead(outputFilePath);

        if (IsValidCropRegion(afterProcessImage, leftRightCropRegion))
        {
            Mat leftRightCroppedImage = new Mat(afterProcessImage, leftRightCropRegion);
            Cv2.ImWrite(outputFilePath, leftRightCroppedImage);
        }
    }
}

void colorImgToGrayscaleImg(string outputFilePath)
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

void imgAntialiasing(string outputFilePath)
{
    using (var src = new Mat(outputFilePath))
    {
        // 가우시안 블러링 적용
        Mat blurred = new Mat();
        Cv2.GaussianBlur(src, blurred, new OpenCvSharp.Size(3, 3), 0);

        // 결과 이미지 저장
        Cv2.ImWrite(outputFilePath, blurred);
    }
}

bool IsValidCropRegion(Mat image, Rect cropRegion)
{
    return cropRegion.X >= 0 && cropRegion.Y >= 0 &&
            cropRegion.X + cropRegion.Width <= image.Width &&
            cropRegion.Y + cropRegion.Height <= image.Height;
}

void PerformPlot(string outputFilePath, PlotInfo plotInfo, string title)
{
    using (PlotEngine plotEngine = PlotFactory.CreatePublishEngine())
    {
        plotEngine.BeginPlot(null, null);
        PlotConfig config = plotInfo.ValidatedConfig;
        config.IsPlotToFile = true;
        plotEngine.BeginDocument(plotInfo, title, null, 1, true, outputFilePath);
        PlotPageInfo plotPageInfo = new PlotPageInfo();
        plotEngine.BeginPage(plotPageInfo, plotInfo, true, null);
        plotEngine.BeginGenerateGraphics(null);

        plotEngine.EndGenerateGraphics(null);
        plotEngine.EndPage(null);
        plotEngine.EndDocument(null);
        plotEngine.EndPlot(null);
        plotEngine.Destroy();
    }
}
```

#### 리본 메뉴 생성하기

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

#### 소켓 전송 예제 (클라이언트)

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

#### SFTP 전송 예제 (클라이언트)

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

#### 소켓 전송 예제 (서버)

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

#### 시스템 변수 설정

```cs
// Get the current value from a system variable
int nMaxSort = System.Convert.ToInt32(Application.GetSystemVariable("MAXSORT"));

// Set system variable to new value
Application.SetSystemVariable("MAXSORT", 100);
```

#### 프롬프트에서 문자열 입력 받기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

PromptStringOptions pStrOpts = new PromptStringOptions("\nEnter your name: ");
pStrOpts.AllowSpaces = true;    // 공백 허용
PromptResult pStrRes = acDoc.Editor.GetString(pStrOpts);    // 사용자에게 문자열 입력 요청

Application.ShowAlertDialog("The name entered was: " + pStrRes.StringResult);
```

#### 프롬프트에서 키워드 입력 받기

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

#### 프롬프트에서 정수, 키워드 혼합 입력 받기

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

#### 프롬프트에서 명령어 호출하기

```cs
Document acDoc = Application.DocumentManager.MdiActiveDocument;

// 마지막의 whitespace는 Enter와 같음
acDoc.SendStringToExecute("._circle 2,2,0 4 ", true, false, false);    // 중심이 (2, 2, 0)이고 반지름이 4인 원
acDoc.SendStringToExecute("._zoom _all ", true, false, false);         // 모든 것이 보이도록 Zoom
```

#### 오브젝트 그리기
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

#### 영역 (Regions) 생성
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

#### 해치 (Hatches) 생성
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

#### 선택(Selection)하기
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

#### 오브젝트 편집하기
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

#### 프로퍼티 (레이어) 조작하기
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

#### 프로퍼티 (컬러) 조작하기
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

#### 프로퍼티 (라인타입) 조작하기
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

#### 레이어 상태 저장/복구
  - 도면에 저장된 레이어 상태 리스팅
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        LayerStateManager acLyrStMan;
        acLyrStMan = acCurDb.LayerStateManager;

        DBDictionary acDbDict;
        acDbDict = acTrans.GetObject(acLyrStMan.LayerStatesDictionaryId(true), OpenMode.ForRead) as DBDictionary;

        string sLayerStateNames = "";

        foreach (DBDictionaryEntry acDbDictEnt in acDbDict)
        {
            sLayerStateNames = sLayerStateNames + "\n" + acDbDictEnt.Key;
        }

        Application.ShowAlertDialog("The saved layer settings in this drawing are:" + sLayerStateNames);

        // Dispose of the transaction
    }
    ```
  - 레이어 상태 저장하기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;
 
    string sLyrStName = "ColorLinetype";
 
    if (acLyrStMan.HasLayerState(sLyrStName) == false)
    {
        acLyrStMan.SaveLayerState(sLyrStName,
                                  LayerStateMasks.Color | 
                                  LayerStateMasks.LineType,
                                  ObjectId.Null);
    }
    ```
  - 레이어 상태 이름 변경하기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;
 
    string sLyrStName = "ColorLinetype";
    string sLyrStNewName = "OldColorLinetype";
 
    if (acLyrStMan.HasLayerState(sLyrStName) == true &&
        acLyrStMan.HasLayerState(sLyrStNewName) == false)
    {
        acLyrStMan.RenameLayerState(sLyrStName, sLyrStNewName);
    }
    ```
  - 레이어 상태 제거하기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;

    string sLyrStName = "ColorLinetype";

    if (acLyrStMan.HasLayerState(sLyrStName) == true)
    {
        acLyrStMan.DeleteLayerState(sLyrStName);
    }
    ```
  - 레이어 상태 복구하기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;
 
    string sLyrStName = "ColorLinetype";
 
    if (acLyrStMan.HasLayerState(sLyrStName) == true)
    {
        acLyrStMan.RestoreLayerState(sLyrStName,                 // 복구하고자 하는 레이어 상태의 이름
                                     ObjectId.Null,              // 복구하고자 하는 레이어 설정이 있는 뷰포트의 오브젝트 ID
                                     1,                          // 0 - 레이어 상태에 속하지 않은 레이어는 변경되지 않음, 1 - 레이어 상태에 속하지 않은 레이어는 꺼짐, 2 - 현재 뷰포트에서 레이어 상태에 속하지 않은 레이어는 동결됨, 4 - 레이어 설정이 뷰포트 오버라이드로 복원됨
                                     LayerStateMasks.Color |
                                     LayerStateMasks.LineType);  // 마스크 값: 어떤 레이어 설정이 복원될지 결정함
    }
    ```
  - 저장된 레이어 상태 내보내기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;

    string sLyrStName = "ColorLinetype";

    if (acLyrStMan.HasLayerState(sLyrStName) == true)
    {
        acLyrStMan.ExportLayerState(sLyrStName, "c:\\my documents\\" + sLyrStName + ".las");
    }
    ```
  - 저장된 레이어 상태 가져오기
    ```cs
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    LayerStateManager acLyrStMan;
    acLyrStMan = acDoc.Database.LayerStateManager;

    string sLyrStFileName = "c:\\my documents\\ColorLinetype.las";

    if (System.IO.File.Exists(sLyrStFileName))
    {
        try
        {
            acLyrStMan.ImportLayerState(sLyrStFileName);
        }
        catch (Autodesk.AutoCAD.Runtime.Exception ex)
        {
            Application.ShowAlertDialog(ex.Message);
        }
    }
    ```

#### 도면에 텍스트 추가하기
  - 멀티라인 텍스트 추가하기
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

        // Create a multiline text object
        using (MText acMText = new MText())
        {
            acMText.Location = new Point3d(2, 2, 0);
            acMText.Width = 4;
            acMText.Contents = "This is a text string for the MText object.";

            acBlkTblRec.AppendEntity(acMText);
            acTrans.AddNewlyCreatedDBObject(acMText, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 멀티라인 텍스트 포맷
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

        // Create a multiline text object
        using (MText acMText = new MText())
        {
            acMText.Location = new Point3d(2, 2, 0);
            acMText.Width = 4.5;
            acMText.Contents = "{{\\H1.5x; Big text}\\A2; over text\\A1;/\\A0;under text}";

            acBlkTblRec.AppendEntity(acMText);
            acTrans.AddNewlyCreatedDBObject(acMText, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 싱글라인 텍스트 생성하기
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

        // Create a single-line text object
        using (DBText acText = new DBText())
        {
            acText.Position = new Point3d(2, 2, 0);
            acText.Height = 0.5;
            acText.TextString = "Hello, World.";

            acBlkTblRec.AppendEntity(acText);
            acTrans.AddNewlyCreatedDBObject(acText, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 기울임 각도 설정하기
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

        // Create a single-line text object
        using (DBText acText = new DBText())
        {
            acText.Position = new Point3d(3, 3, 0);
            acText.Height = 0.5;
            acText.TextString = "Hello, World.";

            // Change the oblique angle of the text object to 45 degrees(0.707 in radians)
            acText.Oblique = 0.707;

            acBlkTblRec.AppendEntity(acText);
            acTrans.AddNewlyCreatedDBObject(acText, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 텍스트 정렬하기
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

        string[] textString = new string[3];
        textString[0] = "Left";
        textString[1] = "Center";
        textString[2] = "Right";

        int[] textAlign = new int[3];
        textAlign[0] = (int)TextHorizontalMode.TextLeft;
        textAlign[1] = (int)TextHorizontalMode.TextCenter;
        textAlign[2] = (int)TextHorizontalMode.TextRight;

        Point3d acPtIns = new Point3d(3, 3, 0);
        Point3d acPtAlign = new Point3d(3, 3, 0);

        int nCnt = 0;

        foreach (string strVal in textString)
        {
            // Create a single-line text object
            using (DBText acText = new DBText())
            {
                acText.Position = acPtIns;
                acText.Height = 0.5;
                acText.TextString = strVal;

                // Set the alignment for the text
                acText.HorizontalMode = (TextHorizontalMode)textAlign[nCnt];

                if (acText.HorizontalMode != TextHorizontalMode.TextLeft)
                {
                    acText.AlignmentPoint = acPtAlign;
                }

                acBlkTblRec.AppendEntity(acText);
                acTrans.AddNewlyCreatedDBObject(acText, true);
            }

            // Create a point over the alignment point of the text
            using (DBPoint acPoint = new DBPoint(acPtAlign))
            {
                acPoint.ColorIndex = 1;

                acBlkTblRec.AppendEntity(acPoint);
                acTrans.AddNewlyCreatedDBObject(acPoint, true);

                // Adjust the insertion and alignment points
                acPtIns = new Point3d(acPtIns.X, acPtIns.Y + 3, 0);
                acPtAlign = acPtIns;
            }

            nCnt = nCnt + 1;
        }

        // Set the point style to crosshair
        Application.SetSystemVariable("PDMODE", 2);

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 텍스트 생성 플래그 설정하기
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

        // Create a single-line text object
        using (DBText acText = new DBText())
        {
            acText.Position = new Point3d(3, 3, 0);
            acText.Height = 0.5;
            acText.TextString = "Hello, World.";

            // Display the text backwards
            acText.IsMirroredInX = true;

            acBlkTblRec.AppendEntity(acText);
            acTrans.AddNewlyCreatedDBObject(acText, true);
        }

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 글꼴 할당하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;
 
    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the current text style for write
        TextStyleTableRecord acTextStyleTblRec;
        acTextStyleTblRec = acTrans.GetObject(acCurDb.Textstyle, OpenMode.ForWrite) as TextStyleTableRecord;
 
        // Get the current font settings
        Autodesk.AutoCAD.GraphicsInterface.FontDescriptor acFont;
        acFont = acTextStyleTblRec.Font;
 
        // Update the text style's typeface with "PlayBill"
        Autodesk.AutoCAD.GraphicsInterface.FontDescriptor acNewFont;
        acNewFont = new
          Autodesk.AutoCAD.GraphicsInterface.FontDescriptor("PlayBill",
                                                            acFont.Bold,
                                                            acFont.Italic,
                                                            acFont.CharacterSet,
                                                            acFont.PitchAndFamily);
 
        acTextStyleTblRec.Font = acNewFont;
 
        acDoc.Editor.Regen();
 
        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - Unicode 및 큰 글꼴 사용하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the current text style for write
        TextStyleTableRecord acTextStyleTblRec;
        acTextStyleTblRec = acTrans.GetObject(acCurDb.Textstyle, OpenMode.ForWrite) as TextStyleTableRecord;

        // Change the font files used for both Big and Regular fonts
        acTextStyleTblRec.BigFontFileName = "C:/AutoCAD/Fonts/bigfont.shx";
        acTextStyleTblRec.FileName = "C:/AutoCAD/Fonts/italic.shx";

        // Save the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```

#### 치수선 및 공차
  - 선형 치수선 생성
    ```cs
    // Get the current database
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

        // Create the rotated dimension
        using (RotatedDimension acRotDim = new RotatedDimension())
        {
            acRotDim.XLine1Point = new Point3d(0, 0, 0);
            acRotDim.XLine2Point = new Point3d(6, 3, 0);
            acRotDim.Rotation = 0.707;
            acRotDim.DimLinePoint = new Point3d(0, 5, 0);
            acRotDim.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acRotDim);
            acTrans.AddNewlyCreatedDBObject(acRotDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 원형 치수선 생성
    ```cs
    // 원/호 크기, 치수선 텍스트 위치, 시스템 변수의 값에 따라 여러 가지 타입의 원형 치수선 타입이 생성됨
    // 치수 관련 시스템 변수는 다음과 같다: DIMUPT, DIMTOFL, DIMFIT, DIMTIH, DIMTOH, DIMJUST, DIMTAD
    
    // Get the current database
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

        // Create the radial dimension
        using (RadialDimension acRadDim = new RadialDimension())
        {
            acRadDim.Center = new Point3d(0, 0, 0);
            acRadDim.ChordPoint = new Point3d(5, 5, 0);
            acRadDim.LeaderLength = 5;
            acRadDim.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acRadDim);
            acTrans.AddNewlyCreatedDBObject(acRadDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 각도 치수선 생성
    ```cs
    // 각도 치수선을 생성하기 위한 2가지 오브젝트가 있음
    // LineAngularDimension2: 2개의 라인에 의해 정의됨
    // Point3AngularDimension: 3개의 점에 의해 정의됨
    
    // Get the current database
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

        // Create an angular dimension
        using (LineAngularDimension2 acLinAngDim = new LineAngularDimension2())
        {
            acLinAngDim.XLine1Start = new Point3d(0, 5, 0);
            acLinAngDim.XLine1End = new Point3d(1, 7, 0);
            acLinAngDim.XLine2Start = new Point3d(0, 5, 0);
            acLinAngDim.XLine2End = new Point3d(1, 3, 0);
            acLinAngDim.ArcPoint = new Point3d(3, 5, 0);
            acLinAngDim.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acLinAngDim);
            acTrans.AddNewlyCreatedDBObject(acLinAngDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 조그 원형 치수선 생성
    ```cs
    // Get the current database
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

        // Create a large radius dimension
        using (RadialDimensionLarge acRadDimLrg = new RadialDimensionLarge())
        {
            acRadDimLrg.Center = new Point3d(-3, -4, 0);
            acRadDimLrg.ChordPoint = new Point3d(2, 7, 0);
            acRadDimLrg.OverrideCenter = new Point3d(0, 2, 0);
            acRadDimLrg.JogPoint = new Point3d(1, 4.5, 0);
            acRadDimLrg.JogAngle = 0.707;
            acRadDimLrg.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acRadDimLrg);
            acTrans.AddNewlyCreatedDBObject(acRadDimLrg, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 호 길이 치수선 생성
    ```cs
    // Get the current database
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

        // Create an arc length dimension
        using (ArcDimension acArcDim = new ArcDimension(new Point3d(4.5, 1.5, 0),
                                                        new Point3d(8, 4.25, 0),
                                                        new Point3d(0, 2, 0),
                                                        new Point3d(5, 7, 0),
                                                        "<>",
                                                        acCurDb.Dimstyle))
        {

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acArcDim);
            acTrans.AddNewlyCreatedDBObject(acArcDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 세로 치수선 생성
    ```cs
    // Get the current database
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

        // Create an ordinate dimension
        using (OrdinateDimension acOrdDim = new OrdinateDimension())
        {
            acOrdDim.UsingXAxis = true;
            acOrdDim.DefiningPoint = new Point3d(5, 5, 0);
            acOrdDim.LeaderEndPoint = new Point3d(10, 5, 0);
            acOrdDim.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acOrdDim);
            acTrans.AddNewlyCreatedDBObject(acOrdDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 치수 텍스트 오버라이드
    ```cs
    // Get the current database
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

        // Create the aligned dimension
        using (AlignedDimension acAliDim = new AlignedDimension())
        {
            acAliDim.XLine1Point = new Point3d(5, 3, 0);
            acAliDim.XLine2Point = new Point3d(10, 3, 0);
            acAliDim.DimLinePoint = new Point3d(7.5, 5, 0);
            acAliDim.DimensionStyle = acCurDb.Dimstyle;

            // Override the dimension text
            acAliDim.DimensionText = "The value is <>";

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acAliDim);
            acTrans.AddNewlyCreatedDBObject(acAliDim, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 치수선 스타일 생성/변경/복사
    ```cs
    // Get the current database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for read
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForRead) as BlockTableRecord;

        object acObj = null;
        foreach (ObjectId acObjId in acBlkTblRec)
        {
            // Get the first object in Model space
            acObj = acTrans.GetObject(acObjId, OpenMode.ForRead);

            break;
        }

        // Open the DimStyle table for read
        DimStyleTable acDimStyleTbl;
        acDimStyleTbl = acTrans.GetObject(acCurDb.DimStyleTableId, OpenMode.ForRead) as DimStyleTable;

        string[] strDimStyleNames = new string[3];
        strDimStyleNames[0] = "Style 1 copied from a dim";
        strDimStyleNames[1] = "Style 2 copied from Style 1";
        strDimStyleNames[2] = "Style 3 copied from the running drawing values";

        int nCnt = 0;

        // Keep a reference of the first dimension style for later
        DimStyleTableRecord acDimStyleTblRec1 = null;

        // Iterate the array of dimension style names
        foreach (string strDimStyleName in strDimStyleNames)
        {
            DimStyleTableRecord acDimStyleTblRec;
            DimStyleTableRecord acDimStyleTblRecCopy = null;

            // Check to see if the dimension style exists or not
            if (acDimStyleTbl.Has(strDimStyleName) == false)
            {
                if (acDimStyleTbl.IsWriteEnabled == false) acTrans.GetObject(acCurDb.DimStyleTableId, OpenMode.ForWrite);

                acDimStyleTblRec = new DimStyleTableRecord();
                acDimStyleTblRec.Name = strDimStyleName;

                acDimStyleTbl.Add(acDimStyleTblRec);
                acTrans.AddNewlyCreatedDBObject(acDimStyleTblRec, true);
            }
            else
            {
                acDimStyleTblRec = acTrans.GetObject(acDimStyleTbl[strDimStyleName], OpenMode.ForWrite) as DimStyleTableRecord;
            }

            // Determine how the new dimension style is populated
            switch ((int)nCnt)
            {
                // Assign the values of the dimension object to the new dimension style
                case 0:
                    try
                    {
                        // Cast the object to a Dimension
                        Dimension acDim = acObj as Dimension;

                        // Copy the dimension style data from the dimension and
                        // set the name of the dimension style as the copied settings
                        // are unnamed.
                        acDimStyleTblRecCopy = acDim.GetDimstyleData();
                        acDimStyleTblRec1 = acDimStyleTblRec;
                    }
                    catch
                    {
                        // Object was not a dimension
                    }

                    break;

                // Assign the values of the dimension style to the new dimension style
                case 1:
                    acDimStyleTblRecCopy = acDimStyleTblRec1;
                    break;
                // Assign the values of the current drawing to the dimension style
                case 2:
                    acDimStyleTblRecCopy = acCurDb.GetDimstyleData();
                    break;
            }

            // Copy the dimension settings and set the name of the dimension style
            acDimStyleTblRec.CopyFrom(acDimStyleTblRecCopy);
            acDimStyleTblRec.Name = strDimStyleName;

            // Dispose of the copied dimension style
            acDimStyleTblRecCopy.Dispose();

            nCnt = nCnt + 1;
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 치수선 스타일 오버라이드
    ```cs
    // Get the current database
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

        // Create the aligned dimension
        using (AlignedDimension acAliDim = new AlignedDimension())
        {
            acAliDim.XLine1Point = new Point3d(0, 5, 0);
            acAliDim.XLine2Point = new Point3d(5, 5, 0);
            acAliDim.DimLinePoint = new Point3d(5, 7, 0);
            acAliDim.DimensionStyle = acCurDb.Dimstyle;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acAliDim);
            acTrans.AddNewlyCreatedDBObject(acAliDim, true);

            // Append a suffix to the dimension text
            PromptStringOptions pStrOpts = new PromptStringOptions("");

            pStrOpts.Message = "\nEnter a new text suffix for the dimension: ";
            pStrOpts.AllowSpaces = true;
            PromptResult pStrRes = acDoc.Editor.GetString(pStrOpts);

            if (pStrRes.Status == PromptStatus.OK)
            {
                acAliDim.Suffix = pStrRes.StringResult;
            }
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 지시선 생성
    ```cs
    // Get the current database
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

        // Create the leader
        using (Leader acLdr = new Leader())
        {
            acLdr.AppendVertex(new Point3d(0, 0, 0));
            acLdr.AppendVertex(new Point3d(4, 4, 0));
            acLdr.AppendVertex(new Point3d(4, 5, 0));
            acLdr.HasArrowHead = true;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acLdr);
            acTrans.AddNewlyCreatedDBObject(acLdr, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 지시선 연관성
    ```cs
    // Get the current database
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

        // Create the MText annotation
        using (MText acMText = new MText())
        {
            acMText.Contents = "Hello, World.";
            acMText.Location = new Point3d(5, 5, 0);
            acMText.Width = 2;

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acMText);
            acTrans.AddNewlyCreatedDBObject(acMText, true);

            // Create the leader with annotation
            using (Leader acLdr = new Leader())
            {
                acLdr.AppendVertex(new Point3d(0, 0, 0));
                acLdr.AppendVertex(new Point3d(4, 4, 0));
                acLdr.AppendVertex(new Point3d(4, 5, 0));
                acLdr.HasArrowHead = true;

                // Add the new object to Model space and the transaction
                acBlkTblRec.AppendEntity(acLdr);
                acTrans.AddNewlyCreatedDBObject(acLdr, true);

                // Attach the annotation after the leader object is added
                acLdr.Annotation = acMText.ObjectId;
                acLdr.EvaluateLeader();
            }
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```
  - 기하학적 공차 생성
    ```cs
    // Get the current database
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

        // Create the Geometric Tolerance (Feature Control Frame)
        using (FeatureControlFrame acFcf = new FeatureControlFrame())
        {
            acFcf.Text = "{\\Fgdt;j}%%v{\\Fgdt;n}0.001%%v%%v%%v%%v";
            acFcf.Location = new Point3d(5, 5, 0);

            // Add the new object to Model space and the transaction
            acBlkTblRec.AppendEntity(acFcf);
            acTrans.AddNewlyCreatedDBObject(acFcf, true);
        }

        // Commit the changes and dispose of the transaction
        acTrans.Commit();
    }
    ```

#### 3D 공간
  - 3D 좌표 지정하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a polyline with two segments (3 points)
        using (Polyline acPoly = new Polyline())
        {
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.ColorIndex = 1;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);


            // Create a 3D polyline with two segments (3 points)
            using (Polyline3d acPoly3d = new Polyline3d())
            {
                acPoly3d.ColorIndex = 5;

                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acPoly3d);
                acTrans.AddNewlyCreatedDBObject(acPoly3d, true);

                // Before adding vertexes, the polyline must be in the drawing
                Point3dCollection acPts3dPoly = new Point3dCollection();
                acPts3dPoly.Add(new Point3d(1, 1, 0));
                acPts3dPoly.Add(new Point3d(2, 1, 0));
                acPts3dPoly.Add(new Point3d(2, 2, 0));

                foreach (Point3d acPt3d in acPts3dPoly)
                {
                    using (PolylineVertex3d acPolVer3d = new PolylineVertex3d(acPt3d))
                    {
                        acPoly3d.AppendVertex(acPolVer3d);
                        acTrans.AddNewlyCreatedDBObject(acPolVer3d, true);
                    }
                }

                // Get the coordinates of the lightweight polyline
                Point2dCollection acPts2d = new Point2dCollection();
                for (int nCnt = 0; nCnt < acPoly.NumberOfVertices; nCnt++)
                {
                    acPts2d.Add(acPoly.GetPoint2dAt(nCnt));
                }

                // Get the coordinates of the 3D polyline
                Point3dCollection acPts3d = new Point3dCollection();
                foreach (ObjectId acObjIdVert in acPoly3d)
                {
                    PolylineVertex3d acPolVer3d;
                    acPolVer3d = acTrans.GetObject(acObjIdVert, OpenMode.ForRead) as PolylineVertex3d;

                    acPts3d.Add(acPolVer3d.Position);
                }

                // Display the Coordinates
                Application.ShowAlertDialog("2D polyline (red): \n" +
                                            acPts2d[0].ToString() + "\n" +
                                            acPts2d[1].ToString() + "\n" +
                                            acPts2d[2].ToString());

                Application.ShowAlertDialog("3D polyline (blue): \n" +
                                            acPts3d[0].ToString() + "\n" +
                                            acPts3d[1].ToString() + "\n" +
                                            acPts3d[2].ToString());
            }
        }

        // Save the new object to the database
        acTrans.Commit();
    }
    ```
  - 사용자 좌표계 (UCS) 정의하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the UCS table for read
        UcsTable acUCSTbl;
        acUCSTbl = acTrans.GetObject(acCurDb.UcsTableId, OpenMode.ForRead) as UcsTable;

        UcsTableRecord acUCSTblRec;

        // Check to see if the "New_UCS" UCS table record exists
        if (acUCSTbl.Has("New_UCS") == false)
        {
            acUCSTblRec = new UcsTableRecord();
            acUCSTblRec.Name = "New_UCS";

            // Open the UCSTable for write
            acTrans.GetObject(acCurDb.UcsTableId, OpenMode.ForWrite);

            // Add the new UCS table record
            acUCSTbl.Add(acUCSTblRec);
            acTrans.AddNewlyCreatedDBObject(acUCSTblRec, true);
        }
        else
        {
            acUCSTblRec = acTrans.GetObject(acUCSTbl["New_UCS"], OpenMode.ForWrite) as UcsTableRecord;
        }

        acUCSTblRec.Origin = new Point3d(4, 5, 3);
        acUCSTblRec.XAxis = new Vector3d(1, 0, 0);
        acUCSTblRec.YAxis = new Vector3d(0, 1, 0);

        // Open the active viewport
        ViewportTableRecord acVportTblRec;
        acVportTblRec = acTrans.GetObject(acDoc.Editor.ActiveViewportId, OpenMode.ForWrite) as ViewportTableRecord;

        // Display the UCS Icon at the origin of the current viewport
        acVportTblRec.IconAtOrigin = true;
        acVportTblRec.IconEnabled = true;

        // Set the UCS current
        acVportTblRec.SetUcs(acUCSTblRec.ObjectId);
        acDoc.Editor.UpdateTiledViewportsFromDatabase();

        // Display the name of the current UCS
        UcsTableRecord acUCSTblRecActive;
        acUCSTblRecActive = acTrans.GetObject(acVportTblRec.UcsName, OpenMode.ForRead) as UcsTableRecord;

        Application.ShowAlertDialog("The current UCS is: " + acUCSTblRecActive.Name);

        PromptPointResult pPtRes;
        PromptPointOptions pPtOpts = new PromptPointOptions("");

        // Prompt for a point
        pPtOpts.Message = "\nEnter a point: ";
        pPtRes = acDoc.Editor.GetPoint(pPtOpts);

        Point3d pPt3dWCS;
        Point3d pPt3dUCS;

        // If a point was entered, then translate it to the current UCS
        if (pPtRes.Status == PromptStatus.OK)
        {
            pPt3dWCS = pPtRes.Value;
            pPt3dUCS = pPtRes.Value;

            // Translate the point from the current UCS to the WCS
            Matrix3d newMatrix = new Matrix3d();
            newMatrix = Matrix3d.AlignCoordinateSystem(Point3d.Origin,
                                                        Vector3d.XAxis,
                                                        Vector3d.YAxis,
                                                        Vector3d.ZAxis,
                                                        acVportTblRec.Ucs.Origin,
                                                        acVportTblRec.Ucs.Xaxis,
                                                        acVportTblRec.Ucs.Yaxis,
                                                        acVportTblRec.Ucs.Zaxis);

            pPt3dWCS = pPt3dWCS.TransformBy(newMatrix);

            Application.ShowAlertDialog("The WCS coordinates are: \n" +
                                        pPt3dWCS.ToString() + "\n" +
                                        "The UCS coordinates are: \n" +
                                        pPt3dUCS.ToString());
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 좌표 변환
    ```cs
    // 참고 사항: 좌표계의 종류
    // WCS (World Coordinate System): 다른 모든 좌표계가 이것을 참조하며 이것은 변하지 않는다. 별도로 명시하지 않는 한 .NET API에 입력하는 모든 좌표는 이것을 의미함
    // UCS (User Coordinate System): 사용자 작업 좌표계로, 사용자가 그리기 작업을 할 때 이것을 사용함. AutoCAD 커맨드나 AutoLISP 루틴은 이 좌표계를 사용함
    // OCS (Object Coordinate System): 오브젝트에 대한 메서드, 프로퍼티에 의해 지정된 좌표 값으로 오브젝트의 위치에 상대적임.
    // DCS (Display Coordinate System): 오브젝트가 표시되기 전에 변환되는 좌표계이며 이것의 원점은 AutoCAD 시스템 변수 TARGET에 저장되고 Z축은 뷰잉 방향과 일치한다.
    // PSDCS (Paper space DCS): DCS는 모델 뷰에 대한 좌표계인 반면, PSDCS는 페이퍼 뷰에 대한 좌표계이다. X, Y축에 대한 축척(Scale)이 반드시 반영된다.

    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a 2D polyline with two segments (3 points)
        using (Polyline2d acPoly2d = new Polyline2d())
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly2d);
            acTrans.AddNewlyCreatedDBObject(acPoly2d, true);

            // Before adding vertexes, the polyline must be in the drawing
            Point3dCollection acPts2dPoly = new Point3dCollection();
            acPts2dPoly.Add(new Point3d(1, 1, 0));
            acPts2dPoly.Add(new Point3d(1, 2, 0));
            acPts2dPoly.Add(new Point3d(2, 2, 0));
            acPts2dPoly.Add(new Point3d(3, 2, 0));
            acPts2dPoly.Add(new Point3d(4, 4, 0));

            foreach (Point3d acPt3d in acPts2dPoly)
            {
                Vertex2d acVer2d = new Vertex2d(acPt3d, 0, 0, 0, 0);
                acPoly2d.AppendVertex(acVer2d);
                acTrans.AddNewlyCreatedDBObject(acVer2d, true);
            }

            // Set the normal of the 2D polyline
            acPoly2d.Normal = new Vector3d(0, 1, 2);

            // Get the first coordinate of the 2D polyline
            Point3dCollection acPts3d = new Point3dCollection();
            Vertex2d acFirstVer = null;

            foreach (ObjectId acObjIdVert in acPoly2d)
            {
                acFirstVer = acTrans.GetObject(acObjIdVert, OpenMode.ForRead) as Vertex2d;

                acPts3d.Add(acFirstVer.Position);

                break;
            }

            // Get the first point of the polyline and 
            // use the eleveation for the Z value
            Point3d pFirstVer = new Point3d(acFirstVer.Position.X,
                                            acFirstVer.Position.Y,
                                            acPoly2d.Elevation);

            // Translate the OCS to WCS
            Matrix3d mWPlane = Matrix3d.WorldToPlane(acPoly2d.Normal);
            Point3d pWCSPt = pFirstVer.TransformBy(mWPlane);

            Application.ShowAlertDialog("The first vertex has the following " +
                                        "coordinates:" +
                                        "\nOCS: " + pFirstVer.ToString() +
                                        "\nWCS: " + pWCSPt.ToString());
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 오브젝트 (메시) 생성
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a polygon mesh
        using (PolygonMesh acPolyMesh = new PolygonMesh())
        {
            acPolyMesh.MSize = 4;
            acPolyMesh.NSize = 4;

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPolyMesh);
            acTrans.AddNewlyCreatedDBObject(acPolyMesh, true);

            // Before adding vertexes, the polyline must be in the drawing
            Point3dCollection acPts3dPMesh = new Point3dCollection();
            acPts3dPMesh.Add(new Point3d(0, 0, 0));
            acPts3dPMesh.Add(new Point3d(2, 0, 1));
            acPts3dPMesh.Add(new Point3d(4, 0, 0));
            acPts3dPMesh.Add(new Point3d(6, 0, 1));

            acPts3dPMesh.Add(new Point3d(0, 2, 0));
            acPts3dPMesh.Add(new Point3d(2, 2, 1));
            acPts3dPMesh.Add(new Point3d(4, 2, 0));
            acPts3dPMesh.Add(new Point3d(6, 2, 1));

            acPts3dPMesh.Add(new Point3d(0, 4, 0));
            acPts3dPMesh.Add(new Point3d(2, 4, 1));
            acPts3dPMesh.Add(new Point3d(4, 4, 0));
            acPts3dPMesh.Add(new Point3d(6, 4, 0));

            acPts3dPMesh.Add(new Point3d(0, 6, 0));
            acPts3dPMesh.Add(new Point3d(2, 6, 1));
            acPts3dPMesh.Add(new Point3d(4, 6, 0));
            acPts3dPMesh.Add(new Point3d(6, 6, 0));

            foreach (Point3d acPt3d in acPts3dPMesh)
            {
                PolygonMeshVertex acPMeshVer = new PolygonMeshVertex(acPt3d);
                acPolyMesh.AppendVertex(acPMeshVer);
                acTrans.AddNewlyCreatedDBObject(acPMeshVer, true);
            }
        }

        // Open the active viewport
        ViewportTableRecord acVportTblRec;
        acVportTblRec = acTrans.GetObject(acDoc.Editor.ActiveViewportId, OpenMode.ForWrite) as ViewportTableRecord;

        // Rotate the view direction of the current viewport
        acVportTblRec.ViewDirection = new Vector3d(-1, -1, 1);
        acDoc.Editor.UpdateTiledViewportsFromDatabase();

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 오브젝트 (폴리면 메시) 생성
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a polyface mesh
        using (PolyFaceMesh acPFaceMesh = new PolyFaceMesh())
        {
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPFaceMesh);
            acTrans.AddNewlyCreatedDBObject(acPFaceMesh, true);

            // Before adding vertexes, the polyline must be in the drawing
            Point3dCollection acPts3dPFMesh = new Point3dCollection();
            acPts3dPFMesh.Add(new Point3d(4, 7, 0));
            acPts3dPFMesh.Add(new Point3d(5, 7, 0));
            acPts3dPFMesh.Add(new Point3d(6, 7, 0));

            acPts3dPFMesh.Add(new Point3d(4, 6, 0));
            acPts3dPFMesh.Add(new Point3d(5, 6, 0));
            acPts3dPFMesh.Add(new Point3d(6, 6, 1));

            foreach (Point3d acPt3d in acPts3dPFMesh)
            {
                PolyFaceMeshVertex acPMeshVer = new PolyFaceMeshVertex(acPt3d);
                acPFaceMesh.AppendVertex(acPMeshVer);
                acTrans.AddNewlyCreatedDBObject(acPMeshVer, true);
            }

            using (FaceRecord acFaceRec1 = new FaceRecord(1, 2, 5, 4))
            {
                acPFaceMesh.AppendFaceRecord(acFaceRec1);
                acTrans.AddNewlyCreatedDBObject(acFaceRec1, true);
            }

            using (FaceRecord acFaceRec2 = new FaceRecord(2, 3, 6, 5))
            {
                acPFaceMesh.AppendFaceRecord(acFaceRec2);
                acTrans.AddNewlyCreatedDBObject(acFaceRec2, true);
            }
        }

        // Open the active viewport
        ViewportTableRecord acVportTblRec;
        acVportTblRec = acTrans.GetObject(acDoc.Editor.ActiveViewportId, OpenMode.ForWrite) as ViewportTableRecord;

        // Rotate the view direction of the current viewport
        acVportTblRec.ViewDirection = new Vector3d(-1, -1, 1);
        acDoc.Editor.UpdateTiledViewportsFromDatabase();

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 오브젝트 (솔리드) 생성
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a 3D solid wedge
        using (Solid3d acSol3D = new Solid3d())
        {
            acSol3D.CreateWedge(10, 15, 20);

            // Position the center of the 3D solid at (5,5,0) 
            acSol3D.TransformBy(Matrix3d.Displacement(new Point3d(5, 5, 0) - Point3d.Origin));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSol3D);
            acTrans.AddNewlyCreatedDBObject(acSol3D, true);
        }

        // Open the active viewport
        ViewportTableRecord acVportTblRec;
        acVportTblRec = acTrans.GetObject(acDoc.Editor.ActiveViewportId, OpenMode.ForWrite) as ViewportTableRecord;

        // Rotate the view direction of the current viewport
        acVportTblRec.ViewDirection = new Vector3d(-1, -1, 1);
        acDoc.Editor.UpdateTiledViewportsFromDatabase();

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 회전
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a 3D solid box
        using (Solid3d acSol3D = new Solid3d())
        {
            acSol3D.CreateBox(5, 7, 10);

            // Position the center of the 3D solid at (5,5,0) 
            acSol3D.TransformBy(Matrix3d.Displacement(new Point3d(5, 5, 0) - Point3d.Origin));

            Matrix3d curUCSMatrix = acDoc.Editor.CurrentUserCoordinateSystem;
            CoordinateSystem3d curUCS = curUCSMatrix.CoordinateSystem3d;

            // Rotate the 3D solid 30 degrees around the axis that is
            // defined by the points (-3,4,0) and (-3,-4,0)
            Vector3d vRot = new Point3d(-3, 4, 0).
                            GetVectorTo(new Point3d(-3, -4, 0));

            acSol3D.TransformBy(Matrix3d.Rotation(0.5236,
                                                    vRot,
                                                    new Point3d(-3, 4, 0)));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSol3D);
            acTrans.AddNewlyCreatedDBObject(acSol3D, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 격자 배치
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
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

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acCirc);
            acTrans.AddNewlyCreatedDBObject(acCirc, true);

            // Create a rectangular array with 4 rows, 4 columns, and 3 levels
            int nRows = 4;
            int nColumns = 4;
            int nLevels = 3;

            // Set the row, column, and level offsets along with the base array angle
            double dRowOffset = 1;
            double dColumnOffset = 1;
            double dLevelsOffset = 4;
            double dArrayAng = 0;

            // Get the angle from X for the current UCS 
            Matrix3d curUCSMatrix = acDoc.Editor.CurrentUserCoordinateSystem;
            CoordinateSystem3d curUCS = curUCSMatrix.CoordinateSystem3d;
            Vector2d acVec2dAng = new Vector2d(curUCS.Xaxis.X,
                                               curUCS.Xaxis.Y);

            // If the UCS is rotated, adjust the array angle accordingly
            dArrayAng = dArrayAng + acVec2dAng.Angle;

            // Use the upper-left corner of the objects extents for the array base point
            Extents3d acExts = acCirc.Bounds.GetValueOrDefault();
            Point2d acPt2dArrayBase = new Point2d(acExts.MinPoint.X,
                                                  acExts.MaxPoint.Y);

            // Track the objects created for each column
            DBObjectCollection acDBObjCollCols = new DBObjectCollection();
            acDBObjCollCols.Add(acCirc);

            // Create the number of objects for the first column
            int nColumnsCount = 1;
            while (nColumns > nColumnsCount)
            {
                Entity acEntClone = acCirc.Clone() as Entity;
                acDBObjCollCols.Add(acEntClone);

                // Caclucate the new point for the copied object (move)
                Point2d acPt2dTo = PolarPoints(acPt2dArrayBase,
                                               dArrayAng,
                                               dColumnOffset * nColumnsCount);

                Vector2d acVec2d = acPt2dArrayBase.GetVectorTo(acPt2dTo);
                Vector3d acVec3d = new Vector3d(acVec2d.X, acVec2d.Y, 0);
                acEntClone.TransformBy(Matrix3d.Displacement(acVec3d));

                acBlkTblRec.AppendEntity(acEntClone);
                acTrans.AddNewlyCreatedDBObject(acEntClone, true);

                nColumnsCount = nColumnsCount + 1;
            }

            // Set a value in radians for 90 degrees
            double dAng = 1.5708;

            // Track the objects created for each row and column
            DBObjectCollection acDBObjCollLvls = new DBObjectCollection();

            foreach (DBObject acObj in acDBObjCollCols)
            {
                acDBObjCollLvls.Add(acObj);
            }

            // Create the number of objects for each row
            foreach (Entity acEnt in acDBObjCollCols)
            {
                int nRowsCount = 1;

                while (nRows > nRowsCount)
                {
                    Entity acEntClone = acEnt.Clone() as Entity;
                    acDBObjCollLvls.Add(acEntClone);

                    // Caclucate the new point for the copied object (move)
                    Point2d acPt2dTo = PolarPoints(acPt2dArrayBase,
                                                   dArrayAng + dAng,
                                                   dRowOffset * nRowsCount);

                    Vector2d acVec2d = acPt2dArrayBase.GetVectorTo(acPt2dTo);
                    Vector3d acVec3d = new Vector3d(acVec2d.X, acVec2d.Y, 0);
                    acEntClone.TransformBy(Matrix3d.Displacement(acVec3d));

                    acBlkTblRec.AppendEntity(acEntClone);
                    acTrans.AddNewlyCreatedDBObject(acEntClone, true);

                    nRowsCount = nRowsCount + 1;
                }
            }

            // Create the number of levels for a 3D array
            foreach (Entity acEnt in acDBObjCollLvls)
            {
                int nLvlsCount = 1;

                while (nLevels > nLvlsCount)
                {
                    Entity acEntClone = acEnt.Clone() as Entity;

                    Vector3d acVec3d = new Vector3d(0, 0, dLevelsOffset * nLvlsCount);
                    acEntClone.TransformBy(Matrix3d.Displacement(acVec3d));

                    acBlkTblRec.AppendEntity(acEntClone);
                    acTrans.AddNewlyCreatedDBObject(acEntClone, true);

                    nLvlsCount = nLvlsCount + 1;
                }
            }
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 미러링
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a 3D solid box
        using (Solid3d acSol3D = new Solid3d())
        {
            acSol3D.CreateBox(5, 7, 10);

            // Position the center of the 3D solid at (5,5,0) 
            acSol3D.TransformBy(Matrix3d.Displacement(new Point3d(5, 5, 0) - Point3d.Origin));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSol3D);
            acTrans.AddNewlyCreatedDBObject(acSol3D, true);

            // Create a copy of the original 3D solid and change the color of the copy
            Solid3d acSol3DCopy = acSol3D.Clone() as Solid3d;
            acSol3DCopy.ColorIndex = 1;

            // Define the mirror plane
            Plane acPlane = new Plane(new Point3d(1.25, 0, 0),
                                        new Point3d(1.25, 2, 0),
                                        new Point3d(1.25, 2, 2));

            // Mirror the 3D solid across the plane
            acSol3DCopy.TransformBy(Matrix3d.Mirroring(acPlane));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSol3DCopy);
            acTrans.AddNewlyCreatedDBObject(acSol3DCopy, true);
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```
  - 3D 솔리드 편집 (합/차/교집합 연산 등)
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table record for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        // Open the Block table record Model space for write
        BlockTableRecord acBlkTblRec;
        acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;

        // Create a 3D solid box
        using (Solid3d acSol3DBox = new Solid3d())
        {
            acSol3DBox.CreateBox(5, 7, 10);
            acSol3DBox.ColorIndex = 7;

            // Position the center of the 3D solid at (5,5,0) 
            acSol3DBox.TransformBy(Matrix3d.Displacement(new Point3d(5, 5, 0) - Point3d.Origin));

            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acSol3DBox);
            acTrans.AddNewlyCreatedDBObject(acSol3DBox, true);

            // Create a 3D solid cylinder
            // 3D solids are created at (0,0,0) so there is no need to move it
            using (Solid3d acSol3DCyl = new Solid3d())
            {
                acSol3DCyl.CreateFrustum(20, 5, 5, 5);
                acSol3DCyl.ColorIndex = 4;

                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acSol3DCyl);
                acTrans.AddNewlyCreatedDBObject(acSol3DCyl, true);

                // Create a 3D solid from the interference of the box and cylinder
                Solid3d acSol3DCopy = acSol3DCyl.Clone() as Solid3d;

                // Check to see if the 3D solids overlap
                if (acSol3DCopy.CheckInterference(acSol3DBox) == true)
                {
                    acSol3DCopy.BooleanOperation(BooleanOperationType.BoolIntersect, acSol3DBox.Clone() as Solid3d);

                    acSol3DCopy.ColorIndex = 1;
                }

                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acSol3DCopy);
                acTrans.AddNewlyCreatedDBObject(acSol3DCopy, true);
            }
        }

        // Save the new objects to the database
        acTrans.Commit();
    }
    ```

#### 블록으로 작업하기
  - 블록 정의하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        if (!acBlkTbl.Has("CircleBlock"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlock";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                    acBlkTbl.Add(acBlkTblRec);
                    acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 블록 삽입하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        ObjectId blkRecId = ObjectId.Null;

        if (!acBlkTbl.Has("CircleBlock"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlock";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                    acBlkTbl.Add(acBlkTblRec);
                    acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                }

                blkRecId = acBlkTblRec.Id;
            }
        }
        else
        {
            blkRecId = acBlkTbl["CircleBlock"];
        }

        // Insert the block into the current space
        if (blkRecId != ObjectId.Null)
        {
            using (BlockReference acBlkRef = new BlockReference(new Point3d(0, 0, 0), blkRecId))
            {
                BlockTableRecord acCurSpaceBlkTblRec;
                acCurSpaceBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acCurSpaceBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 블록 레퍼런스 분해하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        ObjectId blkRecId = ObjectId.Null;

        if (!acBlkTbl.Has("CircleBlock"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlock";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                    acBlkTbl.Add(acBlkTblRec);
                    acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                }

                blkRecId = acBlkTblRec.Id;
            }
        }
        else
        {
            blkRecId = acBlkTbl["CircleBlock"];
        }

        // Insert the block into the current space
        if (blkRecId != ObjectId.Null)
        {
            using (BlockReference acBlkRef = new BlockReference(new Point3d(0, 0, 0), blkRecId))
            {
                BlockTableRecord acCurSpaceBlkTblRec;
                acCurSpaceBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acCurSpaceBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                using (DBObjectCollection dbObjCol = new DBObjectCollection())
                {
                    acBlkRef.Explode(dbObjCol);

                    foreach (DBObject dbObj in dbObjCol)
                    {
                        Entity acEnt = dbObj as Entity;

                        acCurSpaceBlkTblRec.AppendEntity(acEnt);
                        acTrans.AddNewlyCreatedDBObject(dbObj, true);

                        acEnt = acTrans.GetObject(dbObj.ObjectId, OpenMode.ForWrite) as Entity;

                        acEnt.ColorIndex = 1;
                        Application.ShowAlertDialog("Exploded Object: " + acEnt.GetRXClass().DxfName);
                    }
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 블록 재정의하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        if (!acBlkTbl.Has("CircleBlock"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlock";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                    acBlkTbl.Add(acBlkTblRec);
                    acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);

                    // Insert the block into the current space
                    using (BlockReference acBlkRef = new BlockReference(new Point3d(0, 0, 0), acBlkTblRec.Id))
                    {
                        BlockTableRecord acModelSpace;
                        acModelSpace = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                        acModelSpace.AppendEntity(acBlkRef);
                        acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                        Application.ShowAlertDialog("CircleBlock has been created.");
                    }
                }
            }
        }
        else
        {
            // Redefine the block if it exists
            BlockTableRecord acBlkTblRec = acTrans.GetObject(acBlkTbl["CircleBlock"], OpenMode.ForWrite) as BlockTableRecord;

            // Step through each object in the block table record
            foreach (ObjectId objID in acBlkTblRec)
            {
                DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                // Revise the circle in the block
                if (dbObj is Circle)
                {
                    Circle acCirc = dbObj as Circle;

                    acTrans.GetObject(objID, OpenMode.ForWrite);
                    acCirc.Radius = acCirc.Radius * 2;
                }
            }

            // Update existing block references
            foreach (ObjectId objID in acBlkTblRec.GetBlockReferenceIds(false, true))
            {
                BlockReference acBlkRef = acTrans.GetObject(objID, OpenMode.ForWrite) as BlockReference;
                acBlkRef.RecordGraphicsModified(true);
            }

            Application.ShowAlertDialog("CircleBlock has been revised.");
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```

#### 애트리뷰트로 작업하기
  - 애트리뷰트 정의 생성하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        if (!acBlkTbl.Has("CircleBlockWithAttributes"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlockWithAttributes";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    // Add an attribute definition to the block
                    using (AttributeDefinition acAttDef = new AttributeDefinition())
                    {
                        acAttDef.Position = new Point3d(0, 0, 0);
                        acAttDef.Verifiable = true;
                        acAttDef.Prompt = "Door #: ";
                        acAttDef.Tag = "Door#";
                        acAttDef.TextString = "DXX";
                        acAttDef.Height = 1;
                        acAttDef.Justify = AttachmentPoint.MiddleCenter;

                        acBlkTblRec.AppendEntity(acAttDef);

                        acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                        acBlkTbl.Add(acBlkTblRec);
                        acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                    }
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 애트리뷰트를 가진 블록 삽입하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        ObjectId blkRecId = ObjectId.Null;

        if (!acBlkTbl.Has("CircleBlockWithAttributes"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlockWithAttributes";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    // Add an attribute definition to the block
                    using (AttributeDefinition acAttDef = new AttributeDefinition())
                    {
                        acAttDef.Position = new Point3d(0, 0, 0);
                        acAttDef.Prompt = "Door #: ";
                        acAttDef.Tag = "Door#";
                        acAttDef.TextString = "DXX";
                        acAttDef.Height = 1;
                        acAttDef.Justify = AttachmentPoint.MiddleCenter;
                        acBlkTblRec.AppendEntity(acAttDef);

                        acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                        acBlkTbl.Add(acBlkTblRec);
                        acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                    }
                }

                blkRecId = acBlkTblRec.Id;
            }
        }
        else
        {
            blkRecId = acBlkTbl["CircleBlockWithAttributes"];
        }

        // Insert the block into the current space
        if (blkRecId != ObjectId.Null)
        {
            BlockTableRecord acBlkTblRec;
            acBlkTblRec = acTrans.GetObject(blkRecId, OpenMode.ForRead) as BlockTableRecord;

            // Create and insert the new block reference
            using (BlockReference acBlkRef = new BlockReference(new Point3d(2, 2, 0), blkRecId))
            {
                BlockTableRecord acCurSpaceBlkTblRec;
                acCurSpaceBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acCurSpaceBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                // Verify block table record has attribute definitions associated with it
                if (acBlkTblRec.HasAttributeDefinitions)
                {
                    // Add attributes from the block table record
                    foreach (ObjectId objID in acBlkTblRec)
                    {
                        DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                        if (dbObj is AttributeDefinition)
                        {
                            AttributeDefinition acAtt = dbObj as AttributeDefinition;

                            if (!acAtt.Constant)
                            {
                                using (AttributeReference acAttRef = new AttributeReference())
                                {
                                    acAttRef.SetAttributeFromBlock(acAtt, acBlkRef.BlockTransform);
                                    acAttRef.Position = acAtt.Position.TransformBy(acBlkRef.BlockTransform);

                                    acAttRef.TextString = acAtt.TextString;

                                    acBlkRef.AttributeCollection.AppendAttribute(acAttRef);

                                    acTrans.AddNewlyCreatedDBObject(acAttRef, true);
                                }
                            }
                        }
                    }
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 애트리뷰트 정의 재정의하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        ObjectId blkRecId = ObjectId.Null;

        if (!acBlkTbl.Has("CircleBlockWithAttributes"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "CircleBlockWithAttributes";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add a circle to the block
                using (Circle acCirc = new Circle())
                {
                    acCirc.Center = new Point3d(0, 0, 0);
                    acCirc.Radius = 2;

                    acBlkTblRec.AppendEntity(acCirc);

                    // Add an attribute definition to the block
                    using (AttributeDefinition acAttDef = new AttributeDefinition())
                    {
                        acAttDef.Position = new Point3d(0, 0, 0);
                        acAttDef.Prompt = "Door #: ";
                        acAttDef.Tag = "Door#";
                        acAttDef.TextString = "DXX";
                        acAttDef.Height = 1;
                        acAttDef.Justify = AttachmentPoint.MiddleCenter;
                        acBlkTblRec.AppendEntity(acAttDef);

                        acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                        acBlkTbl.Add(acBlkTblRec);
                        acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                    }
                }

                blkRecId = acBlkTblRec.Id;
            }
        }
        else
        {
            blkRecId = acBlkTbl["CircleBlockWithAttributes"];
        }

        // Create and insert the new block reference
        if (blkRecId != ObjectId.Null)
        {
            BlockTableRecord acBlkTblRec;
            acBlkTblRec = acTrans.GetObject(blkRecId, OpenMode.ForRead) as BlockTableRecord;

            using (BlockReference acBlkRef = new BlockReference(new Point3d(2, 2, 0), blkRecId))
            {
                BlockTableRecord acCurSpaceBlkTblRec;
                acCurSpaceBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acCurSpaceBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                // Verify block table record has attribute definitions associated with it
                if (acBlkTblRec.HasAttributeDefinitions)
                {
                    // Add attributes from the block table record
                    foreach (ObjectId objID in acBlkTblRec)
                    {
                        DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                        if (dbObj is AttributeDefinition)
                        {
                            AttributeDefinition acAtt = dbObj as AttributeDefinition;

                            if (!acAtt.Constant)
                            {
                                using (AttributeReference acAttRef = new AttributeReference())
                                {
                                    acAttRef.SetAttributeFromBlock(acAtt, acBlkRef.BlockTransform);
                                    acAttRef.Position = acAtt.Position.TransformBy(acBlkRef.BlockTransform);

                                    acAttRef.TextString = acAtt.TextString;

                                    acBlkRef.AttributeCollection.AppendAttribute(acAttRef);
                                    acTrans.AddNewlyCreatedDBObject(acAttRef, true);
                                }
                            }

                            // Change the attribute definition to be displayed as backwards
                            acTrans.GetObject(objID, OpenMode.ForWrite);
                            acAtt.IsMirroredInX = true;
                        }
                    }
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 애트리뷰트 정보 추출하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Open the Block table for read
        BlockTable acBlkTbl;
        acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

        ObjectId blkRecId = ObjectId.Null;

        if (!acBlkTbl.Has("TESTBLOCK"))
        {
            using (BlockTableRecord acBlkTblRec = new BlockTableRecord())
            {
                acBlkTblRec.Name = "TESTBLOCK";

                // Set the insertion point for the block
                acBlkTblRec.Origin = new Point3d(0, 0, 0);

                // Add an attribute definition to the block
                using (AttributeDefinition acAttDef = new AttributeDefinition())
                {
                    acAttDef.Position = new Point3d(5, 5, 0);
                    acAttDef.Prompt = "Attribute Prompt";
                    acAttDef.Tag = "AttributeTag";
                    acAttDef.TextString = "Attribute Value";
                    acAttDef.Height = 1;
                    acAttDef.Justify = AttachmentPoint.MiddleCenter;
                    acBlkTblRec.AppendEntity(acAttDef);

                    acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForWrite);
                    acBlkTbl.Add(acBlkTblRec);
                    acTrans.AddNewlyCreatedDBObject(acBlkTblRec, true);
                }

                blkRecId = acBlkTblRec.Id;
            }
        }
        else
        {
            blkRecId = acBlkTbl["CircleBlockWithAttributes"];
        }

        // Create and insert the new block reference
        if (blkRecId != ObjectId.Null)
        {
            BlockTableRecord acBlkTblRec;
            acBlkTblRec = acTrans.GetObject(blkRecId, OpenMode.ForRead) as BlockTableRecord;

            using (BlockReference acBlkRef = new BlockReference(new Point3d(5, 5, 0), acBlkTblRec.Id))
            {
                BlockTableRecord acCurSpaceBlkTblRec;
                acCurSpaceBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acCurSpaceBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                // Verify block table record has attribute definitions associated with it
                if (acBlkTblRec.HasAttributeDefinitions)
                {
                    // Add attributes from the block table record
                    foreach (ObjectId objID in acBlkTblRec)
                    {
                        DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                        if (dbObj is AttributeDefinition)
                        {
                            AttributeDefinition acAtt = dbObj as AttributeDefinition;

                            if (!acAtt.Constant)
                            {
                                using (AttributeReference acAttRef = new AttributeReference())
                                {
                                    acAttRef.SetAttributeFromBlock(acAtt, acBlkRef.BlockTransform);
                                    acAttRef.Position = acAtt.Position.TransformBy(acBlkRef.BlockTransform);

                                    acAttRef.TextString = acAtt.TextString;

                                    acBlkRef.AttributeCollection.AppendAttribute(acAttRef);
                                    acTrans.AddNewlyCreatedDBObject(acAttRef, true);
                                }
                            }
                        }
                    }

                    // Display the tags and values of the attached attributes
                    string strMessage = "";
                    AttributeCollection attCol = acBlkRef.AttributeCollection;

                    foreach (ObjectId objID in attCol)
                    {
                        DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                        AttributeReference acAttRef = dbObj as AttributeReference;

                        strMessage = strMessage + "Tag: " + acAttRef.Tag + "\n" +
                                        "Value: " + acAttRef.TextString + "\n";

                        // Change the value of the attribute
                        acAttRef.TextString = "NEW VALUE!";
                    }

                    Application.ShowAlertDialog("The attributes for blockReference " + acBlkRef.Name + " are:\n" + strMessage);

                    strMessage = "";
                    foreach (ObjectId objID in attCol)
                    {
                        DBObject dbObj = acTrans.GetObject(objID, OpenMode.ForRead) as DBObject;

                        AttributeReference acAttRef = dbObj as AttributeReference;

                        strMessage = strMessage + "Tag: " + acAttRef.Tag + "\n" +
                                        "Value: " + acAttRef.TextString + "\n";
                    }

                    Application.ShowAlertDialog("The attributes for blockReference " + acBlkRef.Name + " are:\n" + strMessage);
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
 
#### 외부 레퍼런스
  - Xrefs 부착하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - Xrefs 분리하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }

            Application.ShowAlertDialog("The external reference is attached.");

            acCurDb.DetachXref(acXrefId);

            Application.ShowAlertDialog("The external reference is detached.");
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - Xrefs 재로드하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }

            Application.ShowAlertDialog("The external reference is attached.");

            using (ObjectIdCollection acXrefIdCol = new ObjectIdCollection())
            {
                acXrefIdCol.Add(acXrefId);

                acCurDb.ReloadXrefs(acXrefIdCol);
            }

            Application.ShowAlertDialog("The external reference is reloaded.");
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - Xrefs 언로드하기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }

            Application.ShowAlertDialog("The external reference is attached.");

            using (ObjectIdCollection acXrefIdCol = new ObjectIdCollection())
            {
                acXrefIdCol.Add(acXrefId);

                acCurDb.UnloadXrefs(acXrefIdCol);
            }

            Application.ShowAlertDialog("The external reference is unloaded.");
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - Xrefs 바인딩
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);
            }

            Application.ShowAlertDialog("The external reference is attached.");

            using (ObjectIdCollection acXrefIdCol = new ObjectIdCollection())
            {
                acXrefIdCol.Add(acXrefId);

                acCurDb.BindXrefs(acXrefIdCol, false);
            }

            Application.ShowAlertDialog("The external reference is bound.");
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 블록과 Xrefs 클리핑
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Create a reference to a DWG file
        string PathName = "C:\\AutoCAD\\Sample\\Sheet Sets\\Architectural\\Res\\Exterior Elevations.dwg";
        ObjectId acXrefId = acCurDb.AttachXref(PathName, "Exterior Elevations");

        // If a valid reference is created then continue
        if (!acXrefId.IsNull)
        {
            // Attach the DWG reference to the current space
            Point3d insPt = new Point3d(1, 1, 0);
            using (BlockReference acBlkRef = new BlockReference(insPt, acXrefId))
            {
                BlockTableRecord acBlkTblRec;
                acBlkTblRec = acTrans.GetObject(acCurDb.CurrentSpaceId, OpenMode.ForWrite) as BlockTableRecord;

                acBlkTblRec.AppendEntity(acBlkRef);
                acTrans.AddNewlyCreatedDBObject(acBlkRef, true);

                Application.ShowAlertDialog("The external reference is attached.");

                Matrix3d mat = acBlkRef.BlockTransform;
                mat.Inverse();

                Point2dCollection ptCol = new Point2dCollection();

                // Define the first corner of the clipping boundary
                Point3d pt3d = new Point3d(-330, 400, 0);
                pt3d.TransformBy(mat);
                ptCol.Add(new Point2d(pt3d.X, pt3d.Y));

                // Define the second corner of the clipping boundary
                pt3d = new Point3d(1320, 1120, 0);
                pt3d.TransformBy(mat);
                ptCol.Add(new Point2d(pt3d.X, pt3d.Y));

                // Define the normal and elevation for the clipping boundary 
                Vector3d normal;
                double elev = 0;

                if (acCurDb.TileMode == true)
                {
                    normal = acCurDb.Ucsxdir.CrossProduct(acCurDb.Ucsydir);
                    elev = acCurDb.Elevation;
                }
                else
                {
                    normal = acCurDb.Pucsxdir.CrossProduct(acCurDb.Pucsydir);
                    elev = acCurDb.Pelevation;
                }

                // Set the clipping boundary and enable it
                using (Autodesk.AutoCAD.DatabaseServices.Filters.SpatialFilter filter = 
                    new Autodesk.AutoCAD.DatabaseServices.Filters.SpatialFilter())
                {
                    Autodesk.AutoCAD.DatabaseServices.Filters.SpatialFilterDefinition filterDef = 
                        new Autodesk.AutoCAD.DatabaseServices.Filters.SpatialFilterDefinition(ptCol, normal, elev, 0, 0, true);
                    filter.Definition = filterDef;

                    // Define the name of the extension dictionary and entry name
                    string dictName = "ACAD_FILTER";
                    string spName = "SPATIAL";

                    // Check to see if the Extension Dictionary exists, if not create it
                    if (acBlkRef.ExtensionDictionary.IsNull)
                    {
                        acBlkRef.CreateExtensionDictionary();
                    }

                    // Open the Extension Dictionary for write
                    DBDictionary extDict = acTrans.GetObject(acBlkRef.ExtensionDictionary, OpenMode.ForWrite) as DBDictionary;

                    // Check to see if the dictionary for clipped boundaries exists, 
                    // and add the spatial filter to the dictionary
                    if (extDict.Contains(dictName))
                    {
                        DBDictionary filterDict = acTrans.GetObject(extDict.GetAt(dictName), OpenMode.ForWrite) as DBDictionary;

                        if (filterDict.Contains(spName))
                        {
                            filterDict.Remove(spName);
                        }

                        filterDict.SetAt(spName, filter);
                    }
                    else
                    {
                        using (DBDictionary filterDict = new DBDictionary())
                        {
                            extDict.SetAt(dictName, filterDict);

                            acTrans.AddNewlyCreatedDBObject(filterDict, true);
                            filterDict.SetAt(spName, filter);
                        }
                    }

                    // Append the spatial filter to the drawing
                    acTrans.AddNewlyCreatedDBObject(filter, true);
                }
            }

            Application.ShowAlertDialog("The external reference is clipped.");
        }

        // Save the new objects to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```
  - 확장 데이터 할당 및 가져오기
    ```cs
    // Get the current database and start a transaction
    Database acCurDb;
    acCurDb = Application.DocumentManager.MdiActiveDocument.Database;

    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    string appName = "MY_APP";
    string xdataStr = "This is some xdata";

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Request objects to be selected in the drawing area
        PromptSelectionResult acSSPrompt = acDoc.Editor.GetSelection();

        // If the prompt status is OK, objects were selected
        if (acSSPrompt.Status == PromptStatus.OK)
        {
            // Open the Registered Applications table for read
            RegAppTable acRegAppTbl;
            acRegAppTbl = acTrans.GetObject(acCurDb.RegAppTableId, OpenMode.ForRead) as RegAppTable;

            // Check to see if the Registered Applications table record for the custom app exists
            if (acRegAppTbl.Has(appName) == false)
            {
                using (RegAppTableRecord acRegAppTblRec = new RegAppTableRecord())
                {
                    acRegAppTblRec.Name = appName;

                    acTrans.GetObject(acCurDb.RegAppTableId, OpenMode.ForWrite);
                    acRegAppTbl.Add(acRegAppTblRec);
                    acTrans.AddNewlyCreatedDBObject(acRegAppTblRec, true);
                }
            }

            // Define the Xdata to add to each selected object
            using (ResultBuffer rb = new ResultBuffer())
            {
                rb.Add(new TypedValue((int)DxfCode.ExtendedDataRegAppName, appName));
                rb.Add(new TypedValue((int)DxfCode.ExtendedDataAsciiString, xdataStr));

                SelectionSet acSSet = acSSPrompt.Value;

                // Step through the objects in the selection set
                foreach (SelectedObject acSSObj in acSSet)
                {
                    // Open the selected object for write
                    Entity acEnt = acTrans.GetObject(acSSObj.ObjectId,
                                                        OpenMode.ForWrite) as Entity;

                    // Append the extended data to each object
                    acEnt.XData = rb;
                }
            }
        }

        // Save the new object to the database
        acTrans.Commit();

        // Dispose of the transaction
    }
    ```

#### 레이아웃과 플롯
  - 현재 도면에 레이아웃 목록 나열하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Get the layout dictionary of the current database
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        DBDictionary lays = 
            acTrans.GetObject(acCurDb.LayoutDictionaryId, OpenMode.ForRead) as DBDictionary;

        acDoc.Editor.WriteMessage("\nLayouts:");

        // Step through and list each named layout and Model
        foreach (DBDictionaryEntry item in lays)
        {
            acDoc.Editor.WriteMessage("\n  " + item.Key);
        }

        // Abort the changes to the database
        acTrans.Abort();
    }
    ```
  - 현재 도면에 레이아웃 만들기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Get the layout and plot settings of the named pagesetup
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Reference the Layout Manager
        LayoutManager acLayoutMgr = LayoutManager.Current;

        // Create the new layout with default settings
        ObjectId objID = acLayoutMgr.CreateLayout("newLayout");

        // Open the layout
        Layout acLayout = acTrans.GetObject(objID, OpenMode.ForRead) as Layout;

        // Set the layout current if it is not already
        if (acLayout.TabSelected == false)
        {
            acLayoutMgr.CurrentLayout = acLayout.LayoutName;
        }

        // Output some information related to the layout object
        acDoc.Editor.WriteMessage("\nTab Order: " + acLayout.TabOrder +
                                  "\nTab Selected: " + acLayout.TabSelected +
                                  "\nBlock Table Record ID: " +
                                  acLayout.BlockTableRecordId.ToString());

        // Save the changes made
        acTrans.Commit();
    }
    ```
  - 출력 장치의 종이 크기 목록 나열하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    using(PlotSettings plSet = new PlotSettings(true))
    {
        PlotSettingsValidator acPlSetVdr = PlotSettingsValidator.Current;

        // Set the Plotter and page size
        acPlSetVdr.SetPlotConfigurationName(plSet, "DWF6 ePlot.pc3", "ANSI_A_(8.50_x_11.00_Inches)");

        acDoc.Editor.WriteMessage("\nCanonical and Local media names: ");

        int cnt = 0;

        foreach (string mediaName in acPlSetVdr.GetCanonicalMediaNameList(plSet))
        {
            // Output the names of the available media for the specified device
            acDoc.Editor.WriteMessage("\n  " + mediaName + " | " + acPlSetVdr.GetLocaleMediaName(plSet, cnt));

            cnt = cnt + 1;
        }
    }
    ```
  - 출력 장치 목록 나열하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    acDoc.Editor.WriteMessage("\nPlot devices: ");

    foreach (string plotDevice in PlotSettingsValidator.Current.GetPlotDeviceList())
    {
        // Output the names of the available plotter devices
        acDoc.Editor.WriteMessage("\n  " + plotDevice);
    }
    ```
  - 레이아웃 설정
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Reference the Layout Manager
        LayoutManager acLayoutMgr = LayoutManager.Current;

        // Get the current layout and output its name in the Command Line window
        Layout acLayout = acTrans.GetObject(acLayoutMgr.GetLayoutId(acLayoutMgr.CurrentLayout), OpenMode.ForRead) as Layout;

        // Output the name of the current layout and its device
        acDoc.Editor.WriteMessage("\nCurrent layout: " + acLayout.LayoutName);

        acDoc.Editor.WriteMessage("\nCurrent device name: " + acLayout.PlotConfigurationName);

        // Get a copy of the PlotSettings from the layout
        using (PlotSettings acPlSet = new PlotSettings(acLayout.ModelType))
        {
            acPlSet.CopyFrom(acLayout);

            // Update the PlotConfigurationName property of the PlotSettings object
            PlotSettingsValidator acPlSetVdr = PlotSettingsValidator.Current;
            acPlSetVdr.SetPlotConfigurationName(acPlSet, "DWG To PDF.pc3", "ANSI_B_(11.00_x_17.00_Inches)");

            // Zoom to show the whole paper
            acPlSetVdr.SetZoomToPaperOnUpdate(acPlSet, true);

            // Update the layout
            acTrans.GetObject(acLayoutMgr.GetLayoutId(acLayoutMgr.CurrentLayout), OpenMode.ForWrite);
            acLayout.CopyFrom(acPlSet);
        }

        // Output the name of the new device assigned to the layout
        acDoc.Editor.WriteMessage("\nNew device name: " + acLayout.PlotConfigurationName);

        // Save the new objects to the database
        acTrans.Commit();
    }

    // Update the display
    acDoc.Editor.Regen();
    ```
  - 모델/종이 공간 전환하기
    ```cs
    // 시스템 변수를 통해 상태를 확인할 수 있음
    // (TILEMODE == 0) && (CVPORT != 2) : Layout other than Model is active and you are working in Paper space.
    // (TILEMODE == 0) && (CVPORT == 2) : Layout other than Model is active and you are working in a floating viewport.
    // (TILEMODE == 1) && (CVPORT == Any value) : Model layout is active.
    
    // Get the current document
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
    // Get the current values of CVPORT and TILEMODE
    object oCvports = Application.GetSystemVariable("CVPORT");
    object oTilemode = Application.GetSystemVariable("TILEMODE");
 
    // Check to see if the Model layout is active, TILEMODE is 1 when
    // the Model layout is active
    if (System.Convert.ToInt16(oTilemode) == 0)
    {
        // Check to see if Model space is active in a viewport,
        // CVPORT is 2 if Model space is active 
        if (System.Convert.ToInt16(oCvports) == 2)
        {
            acDoc.Editor.SwitchToPaperSpace();
        }
        else
        {
            acDoc.Editor.SwitchToModelSpace();
        }
    }
    else
    {
        // Switch to the previous Paper space layout
        Application.SetSystemVariable("TILEMODE", 0);
    }
    ```
  - 종이 공간 뷰포트 만들기
    ```cs
    [DllImport("acad.exe", CallingConvention = CallingConvention.Cdecl,
     EntryPoint = "?acedSetCurrentVPort@@YA?AW4ErrorStatus@Acad@@PBVAcDbViewport@@@Z")]
    extern static private int acedSetCurrentVPort(IntPtr AcDbVport);
 
    [CommandMethod("CreateFloatingViewport")]
    public static void CreateFloatingViewport()
    {
        // Get the current document and database, and start a transaction
        Document acDoc = Application.DocumentManager.MdiActiveDocument;
        Database acCurDb = acDoc.Database;

        using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
        {
            // Open the Block table for read
            BlockTable acBlkTbl;
            acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;

            // Open the Block table record Paper space for write
            BlockTableRecord acBlkTblRec;
            acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.PaperSpace], OpenMode.ForWrite) as BlockTableRecord;

            // Switch to the previous Paper space layout
            Application.SetSystemVariable("TILEMODE", 0);
            acDoc.Editor.SwitchToPaperSpace();

            // Create a Viewport
            using (Viewport acVport = new Viewport())
            {
                acVport.CenterPoint = new Point3d(3.25, 3, 0);
                acVport.Width = 6;
                acVport.Height = 5;

                // Add the new object to the block table record and the transaction
                acBlkTblRec.AppendEntity(acVport);
                acTrans.AddNewlyCreatedDBObject(acVport, true);

                // Change the view direction
                acVport.ViewDirection = new Vector3d(1, 1, 1);

                // Enable the viewport
                acVport.On = true;

                // Activate model space in the viewport
                acDoc.Editor.SwitchToModelSpace();

                // Set the new viewport current via an imported ObjectARX function
                acedSetCurrentVPort(acVport.UnmanagedObject);
            }

            // Save the new objects to the database
            acTrans.Commit();
        }
    }
    ```
  - 플롯 스타일 나열하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;

    acDoc.Editor.WriteMessage("\nPlot styles: ");

    foreach (string plotStyle in PlotSettingsValidator.Current.GetPlotStyleSheetList())
    {
        // Output the names of the available plot styles
        acDoc.Editor.WriteMessage("\n  " + plotStyle);
    }
    ```
  - 비주얼 스타일 나열하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        DBDictionary vStyles = acTrans.GetObject(acCurDb.VisualStyleDictionaryId, OpenMode.ForRead) as DBDictionary;

        // Output a message to the Command Line history
        acDoc.Editor.WriteMessage("\nVisual styles: ");

        // Step through the dictionary
        foreach (DBDictionaryEntry entry in vStyles)
        {
            // Get the dictionary entry
            DBVisualStyle vStyle = vStyles.GetAt(entry.Key).GetObject(OpenMode.ForRead) as DBVisualStyle;

            // If the visual style is not marked for internal use then output its name
            if (vStyle.InternalUseOnly == false)
            {
                // Output the name of the visual style
                acDoc.Editor.WriteMessage("\n  " + vStyle.Name);
            }
        }
    }
    ```
  - 페이지 설정 나열하기
    ```cs
    // Get the current document and database
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    // Start a transaction
    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        DBDictionary plSettings = acTrans.GetObject(acCurDb.PlotSettingsDictionaryId, OpenMode.ForRead) as DBDictionary;

        acDoc.Editor.WriteMessage("\nPage Setups: ");

        // List each named page setup
        foreach (DBDictionaryEntry item in plSettings)
        {
            acDoc.Editor.WriteMessage("\n  " + item.Key);
        }

        // Abort the changes to the database
        acTrans.Abort();
    }
    ```
  - 페이지 설정 만들기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {

        DBDictionary plSets = acTrans.GetObject(acCurDb.PlotSettingsDictionaryId, OpenMode.ForRead) as DBDictionary;
        DBDictionary vStyles = acTrans.GetObject(acCurDb.VisualStyleDictionaryId, OpenMode.ForRead) as DBDictionary;

        PlotSettings acPlSet = default(PlotSettings);
        bool createNew = false;

        // Reference the Layout Manager
        LayoutManager acLayoutMgr = LayoutManager.Current;

        // Get the current layout and output its name in the Command Line window
        Layout acLayout = acTrans.GetObject(acLayoutMgr.GetLayoutId(acLayoutMgr.CurrentLayout), OpenMode.ForRead) as Layout;

        // Check to see if the page setup exists
        if (plSets.Contains("MyPageSetup") == false)
        {
            createNew = true;

            // Create a new PlotSettings object: 
            //    True - model space, False - named layout
            acPlSet = new PlotSettings(acLayout.ModelType);
            acPlSet.CopyFrom(acLayout);

            acPlSet.PlotSettingsName = "MyPageSetup";
            acPlSet.AddToPlotSettingsDictionary(acCurDb);
            acTrans.AddNewlyCreatedDBObject(acPlSet, true);
        }
        else
        {
            acPlSet = plSets.GetAt("MyPageSetup").GetObject(OpenMode.ForWrite) as PlotSettings;
        }

        // Update the PlotSettings object
        try
        {
            PlotSettingsValidator acPlSetVdr = PlotSettingsValidator.Current;

            // Set the Plotter and page size
            acPlSetVdr.SetPlotConfigurationName(acPlSet, "DWF6 ePlot.pc3", "ANSI_B_(17.00_x_11.00_Inches)");

            // Set to plot to the current display
            if (acLayout.ModelType == false)
            {
                acPlSetVdr.SetPlotType(acPlSet, Autodesk.AutoCAD.DatabaseServices.PlotType.Layout);
            }
            else
            {
                acPlSetVdr.SetPlotType(acPlSet, Autodesk.AutoCAD.DatabaseServices.PlotType.Extents);

                acPlSetVdr.SetPlotCentered(acPlSet, true);
            }

            // Use SetPlotWindowArea with PlotType.Window
            //acPlSetVdr.SetPlotWindowArea(plSet,
            //                             new Extents2d(New Point2d(0.0, 0.0),
            //                             new Point2d(9.0, 12.0)));

            // Use SetPlotViewName with PlotType.View
            //acPlSetVdr.SetPlotViewName(plSet, "MyView");

            // Set the plot offset
            acPlSetVdr.SetPlotOrigin(acPlSet, new Point2d(0, 0));

            // Set the plot scale
            acPlSetVdr.SetUseStandardScale(acPlSet, true);
            acPlSetVdr.SetStdScaleType(acPlSet, StdScaleType.ScaleToFit);
            acPlSetVdr.SetPlotPaperUnits(acPlSet, PlotPaperUnit.Inches);
            acPlSet.ScaleLineweights = true;

            // Specify if plot styles should be displayed on the layout
            acPlSet.ShowPlotStyles = true;

            // Rebuild plotter, plot style, and canonical media lists 
            // (must be called before setting the plot style)
            acPlSetVdr.RefreshLists(acPlSet);

            // Specify the shaded viewport options
            acPlSet.ShadePlot = PlotSettingsShadePlotType.AsDisplayed;

            acPlSet.ShadePlotResLevel = ShadePlotResLevel.Normal;

            // Specify the plot options
            acPlSet.PrintLineweights = true;
            acPlSet.PlotTransparency = false;
            acPlSet.PlotPlotStyles = true;
            acPlSet.DrawViewportsFirst = true;

            // Use only on named layouts - Hide paperspace objects option
            // plSet.PlotHidden = true;

            // Specify the plot orientation
            acPlSetVdr.SetPlotRotation(acPlSet, PlotRotation.Degrees000);

            // Set the plot style
            if (acCurDb.PlotStyleMode == true)
            {
                acPlSetVdr.SetCurrentStyleSheet(acPlSet, "acad.ctb");
            }
            else
            {
                acPlSetVdr.SetCurrentStyleSheet(acPlSet, "acad.stb");
            }

            // Zoom to show the whole paper
            acPlSetVdr.SetZoomToPaperOnUpdate(acPlSet, true);
        }
        catch (Autodesk.AutoCAD.Runtime.Exception es)
        {
            System.Windows.Forms.MessageBox.Show(es.Message);
        }

        // Save the changes made
        acTrans.Commit();

        if (createNew == true)
        {
            acPlSet.Dispose();
        }
    }
    ```
  - 모델 공간 플롯하기
    ```cs
    // Get the current document and database, and start a transaction
    Document acDoc = Application.DocumentManager.MdiActiveDocument;
    Database acCurDb = acDoc.Database;

    using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
    {
        // Reference the Layout Manager
        LayoutManager acLayoutMgr = LayoutManager.Current;

        // Get the current layout and output its name in the Command Line window
        Layout acLayout = acTrans.GetObject(acLayoutMgr.GetLayoutId(acLayoutMgr.CurrentLayout), OpenMode.ForRead) as Layout;

        // Get the PlotInfo from the layout
        using (PlotInfo acPlInfo = new PlotInfo())
        {
            acPlInfo.Layout = acLayout.ObjectId;

            // Get a copy of the PlotSettings from the layout
            using (PlotSettings acPlSet = new PlotSettings(acLayout.ModelType))
            {
                acPlSet.CopyFrom(acLayout);

                // Update the PlotSettings object
                PlotSettingsValidator acPlSetVdr = PlotSettingsValidator.Current;

                // Set the plot type
                acPlSetVdr.SetPlotType(acPlSet, Autodesk.AutoCAD.DatabaseServices.PlotType.Extents);

                // Set the plot scale
                acPlSetVdr.SetUseStandardScale(acPlSet, true);
                acPlSetVdr.SetStdScaleType(acPlSet, StdScaleType.ScaleToFit);

                // Center the plot
                acPlSetVdr.SetPlotCentered(acPlSet, true);

                // Set the plot device to use
                acPlSetVdr.SetPlotConfigurationName(acPlSet, "DWF6 ePlot.pc3", "ANSI_A_(8.50_x_11.00_Inches)");

                // Set the plot info as an override since it will
                // not be saved back to the layout
                acPlInfo.OverrideSettings = acPlSet;

                // Validate the plot info
                using (PlotInfoValidator acPlInfoVdr = new PlotInfoValidator())
                {
                    acPlInfoVdr.MediaMatchingPolicy = MatchingPolicy.MatchEnabled;
                    acPlInfoVdr.Validate(acPlInfo);

                    // Check to see if a plot is already in progress
                    if (PlotFactory.ProcessPlotState == ProcessPlotState.NotPlotting)
                    {
                        using (PlotEngine acPlEng = PlotFactory.CreatePublishEngine())
                        {
                            // Track the plot progress with a Progress dialog
                            using (PlotProgressDialog acPlProgDlg = new PlotProgressDialog(false, 1, true))
                            {
                                using ((acPlProgDlg))
                                {
                                    // Define the status messages to display 
                                    // when plotting starts
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.DialogTitle, "Plot Progress");
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.CancelJobButtonMessage, "Cancel Job");
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.CancelSheetButtonMessage, "Cancel Sheet");
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.SheetSetProgressCaption, "Sheet Set Progress");
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.SheetProgressCaption, "Sheet Progress");

                                    // Set the plot progress range
                                    acPlProgDlg.LowerPlotProgressRange = 0;
                                    acPlProgDlg.UpperPlotProgressRange = 100;
                                    acPlProgDlg.PlotProgressPos = 0;

                                    // Display the Progress dialog
                                    acPlProgDlg.OnBeginPlot();
                                    acPlProgDlg.IsVisible = true;

                                    // Start to plot the layout
                                    acPlEng.BeginPlot(acPlProgDlg, null);

                                    // Define the plot output
                                    acPlEng.BeginDocument(acPlInfo, acDoc.Name, null, 1, true, "c:\\myplot");

                                    // Display information about the current plot
                                    acPlProgDlg.set_PlotMsgString(PlotMessageIndex.Status, "Plotting: " + acDoc.Name + " - " + acLayout.LayoutName);

                                    // Set the sheet progress range
                                    acPlProgDlg.OnBeginSheet();
                                    acPlProgDlg.LowerSheetProgressRange = 0;
                                    acPlProgDlg.UpperSheetProgressRange = 100;
                                    acPlProgDlg.SheetProgressPos = 0;

                                    // Plot the first sheet/layout
                                    using (PlotPageInfo acPlPageInfo = new PlotPageInfo())
                                    {
                                        acPlEng.BeginPage(acPlPageInfo, acPlInfo, true, null);
                                    }

                                    acPlEng.BeginGenerateGraphics(null);
                                    acPlEng.EndGenerateGraphics(null);

                                    // Finish plotting the sheet/layout
                                    acPlEng.EndPage(null);
                                    acPlProgDlg.SheetProgressPos = 100;
                                    acPlProgDlg.OnEndSheet();

                                    // Finish plotting the document
                                    acPlEng.EndDocument(null);

                                    // Finish the plot
                                    acPlProgDlg.PlotProgressPos = 100;
                                    acPlProgDlg.OnEndPlot();
                                    acPlEng.EndPlot(null);
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    ```
  - 레이아웃 발행하기
    ```cs
    using (DsdEntryCollection dsdDwgFiles = new DsdEntryCollection())
    {
        // Define the first layout
        using (DsdEntry dsdDwgFile1 = new DsdEntry())
        {
            // Set the file name and layout
            dsdDwgFile1.DwgName = "C:\\AutoCAD\\Samples\\Sheet Sets\\Architectural\\A-01.dwg";
            dsdDwgFile1.Layout = "MAIN AND SECOND FLOOR PLAN";
            dsdDwgFile1.Title = "A-01 MAIN AND SECOND FLOOR PLAN";

            // Set the page setup override
            dsdDwgFile1.Nps = "";
            dsdDwgFile1.NpsSourceDwg = "";

            dsdDwgFiles.Add(dsdDwgFile1);
        }

        // Define the second layout
        using (DsdEntry dsdDwgFile2 = new DsdEntry())
        {
            // Set the file name and layout
            dsdDwgFile2.DwgName = "C:\\AutoCAD\\Samples\\Sheet Sets\\Architectural\\A-02.dwg";
            dsdDwgFile2.Layout = "ELEVATIONS";
            dsdDwgFile2.Title = "A-02 ELEVATIONS";

            // Set the page setup override
            dsdDwgFile2.Nps = "";
            dsdDwgFile2.NpsSourceDwg = "";

            dsdDwgFiles.Add(dsdDwgFile2);
        }

        // Set the properties for the DSD file and then write it out
        using (DsdData dsdFileData = new DsdData())
        {
            // Set the target information for publishing
            dsdFileData.DestinationName = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments) + "\\MyPublish2.pdf";
            dsdFileData.ProjectPath = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments) + "\\";
            dsdFileData.SheetType = SheetType.MultiPdf;

            // Set the drawings that should be added to the publication
            dsdFileData.SetDsdEntryCollection(dsdDwgFiles);

            // Set the general publishing properties
            dsdFileData.LogFilePath = Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments) + "\\myBatch.txt";

            // Create the DSD file
            dsdFileData.WriteDsd(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments) + "\\batchdrawings2.dsd");

            try
            {
                // Publish the specified drawing files in the DSD file, and
                // honor the behavior of the BACKGROUNDPLOT system variable

                using (DsdData dsdDataFile = new DsdData())
                {
                    dsdDataFile.ReadDsd(Environment.GetFolderPath(Environment.SpecialFolder.MyDocuments) + "\\batchdrawings2.dsd");

                    // Get the DWG to PDF.pc3 and use it as a 
                    // device override for all the layouts
                    PlotConfig acPlCfg = PlotConfigManager.SetCurrentConfig("DWG to PDF.PC3");

                    Application.Publisher.PublishExecute(dsdDataFile, acPlCfg);
                }
            }
            catch (Autodesk.AutoCAD.Runtime.Exception es)
            {
                System.Windows.Forms.MessageBox.Show(es.Message);
            }
        }
    }
    ```

#### 이벤트
  - Application 이벤트 처리하기
    ```cs
    // 이벤트 종류는 다음과 같다.
    // BeginCustomizationMode: Triggered just before AutoCAD enters customization mode.
    // BeginDoubleClick: Triggered when the mouse button is double clicked.
    // BeginQuit: Triggered just before an AutoCAD session ends.
    // DisplayingCustomizeDialog: Triggered just before the Customize dialog box is displayed.
    // DisplayingDraftingSettingsDialog: Triggered just before the Drafting Settings dialog box is displayed.
    // DisplayingOptionDialog: Triggered just before the Options dialog box is displayed.
    // EndCustomizationMode: Triggered when AutoCAD exits customization mode.
    // EnterModal: Triggered just before a modal dialog box is displayed.
    // Idle: Triggered when AutoCAD text.
    // LeaveModal: Triggered when a modal dialog box is closed.
    // PreTranslateMessage: Triggered just before a message is translated by AutoCAD.
    // QuitAborted: Triggered when an attempt to shutdown AutoCAD is aborted.
    // QuitWillStart: Triggered after the BeginQuit event and before shutdown begins.
    // SystemVariableChanged: Triggered when an attempt to change a system variable is made.
    // SystemVariableChanging: Triggered just before an attempt is made at changing a system variable.
    
    [CommandMethod("AddAppEvent")]
    public void AddAppEvent()
    {
        Application.SystemVariableChanged += new Autodesk.AutoCAD.ApplicationServices.SystemVariableChangedEventHandler(appSysVarChanged);
    }

    [CommandMethod("RemoveAppEvent")]
    public void RemoveAppEvent()
    {
        Application.SystemVariableChanged -= new Autodesk.AutoCAD.ApplicationServices.SystemVariableChangedEventHandler(appSysVarChanged);
    }
 
    public void appSysVarChanged(object senderObj, Autodesk.AutoCAD.ApplicationServices.SystemVariableChangedEventArgs sysVarChEvtArgs)
    {
        object oVal = Application.GetSystemVariable(sysVarChEvtArgs.Name);
 
        // Display a message box with the system variable name and the new value
        Application.ShowAlertDialog(sysVarChEvtArgs.Name + " was changed." + "\nNew value: " + oVal.ToString());
    }
    ```
  - DocumentCollection 이벤트 처리하기
    ```cs
    // 이벤트 종류는 다음과 같다.
    // DocumentActivated: Triggered when a document window is activated.
    // DocumentActivationChanged: Triggered after the active document window is deactivated or destroyed.
    // DocumentBecameCurrent: Triggered when a document window is set current and is different from the previous active document window.
    // DocumentCreated: Triggered after a document window is created. Occurs after a new drawing is created or an existing drawing is opened.
    // DocumentCreateStarted: Triggered just before a document window is created. Occurs before a new drawing is created or an existing drawing is opened.
    // DocumentCreationCanceled: Triggered when a request to create a new drawing or to open an existing drawing is cancelled.
    // DocumentDestroyed: Triggered before a document window is destroyed and its associated database object is deleted.
    // DocumentLockModeChanged: Triggered after the lock mode of a document has changed.
    // DocumentLockModeChangeVetoed: Triggered after the lock mode change is vetoed.
    // DocumentLockModeWillChange: Triggered before the lock mode of a document is changed.
    // DocumentToBeActivated: Triggered when a document is about to be activated.
    // DocumentToBeDeactivated: Triggered when a document is about to be deactivated.
    // DocumentToBeDestroyed: Triggered when a document is about to be destroyed.
    
    [CommandMethod("AddDocColEvent")]
    public void AddDocColEvent()
    {
        Application.DocumentManager.DocumentActivated += new DocumentCollectionEventHandler(docColDocAct);
    }

    [CommandMethod("RemoveDocColEvent")]
    public void RemoveDocColEvent()
    {
        Application.DocumentManager.DocumentActivated -= new DocumentCollectionEventHandler(docColDocAct);
    }

    public void docColDocAct(object senderObj, DocumentCollectionEventArgs docColDocActEvtArgs)
    {
        Application.ShowAlertDialog(docColDocActEvtArgs.Document.Name + " was activated.");
    }
    ```
  - Document 이벤트 처리하기
    ```cs
    // 이벤트 종류는 다음과 같다.
    // BeginDocumentClose: Triggered just after a request is received to close a drawing.
    // BeginDwgOpen: Triggered when a drawing is about to be opened.
    // CloseAborted: Triggered when an attempt to close a drawing is aborted.
    // CloseWillStart: Triggered after the BeginDocumentClose event and before closing the drawing begins.
    // CommandCancelled: Triggered when a command is cancelled before it completes.
    // CommandEnded: Triggered immediately after a command completes.
    // CommandFailed: Triggered when a command fails to complete and is not cancelled.
    // CommandWillStart: Triggered immediately after a command is issued, but before it completes.
    // EndDwgOpen: Triggered when a drawing has been opened.
    // ImpliedSelectionChanged: Triggered when the current pickfirst selection set changes.
    // LayoutSwitched: Triggered after a layout has been set current.
    // LayoutSwitching: Triggered after a layout is being set current.
    // LispCancelled: Triggered when the evaluation of a LISP expression is canceled.
    // LispEnded: Triggered upon completion of evaluating a LISP expression.
    // LispWillStart: Triggered immediately after AutoCAD receives a request to evaluate a LISP expression.
    // UnknownCommand: Triggered immediately when an unknown command is entered at the Command prompt.
    // ViewChanged: Triggered after the view of a drawing has changed.
    
    [CommandMethod("AddDocEvent")]
    public void AddDocEvent()
    {
        // Get the current document
        Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
        acDoc.BeginDocumentClose += new DocumentBeginCloseEventHandler(docBeginDocClose);
    }
 
    [CommandMethod("RemoveDocEvent")]
    public void RemoveDocEvent()
    {
        // Get the current document
        Document acDoc = Application.DocumentManager.MdiActiveDocument;
 
        acDoc.BeginDocumentClose -= new DocumentBeginCloseEventHandler(docBeginDocClose);
    }
 
    public void docBeginDocClose(object senderObj, DocumentBeginCloseEventArgs docBegClsEvtArgs)
    {
            // Display a message box prompting to continue closing the document
            if (System.Windows.Forms.MessageBox.Show(
                       "The document is about to be closed." +
                       "\nDo you want to continue?",
                       "Close Document",
                       System.Windows.Forms.MessageBoxButtons.YesNo) ==
                       System.Windows.Forms.DialogResult.No)
        {
            docBegClsEvtArgs.Veto();
        }
    }
    ```
  - Object 이벤트 처리하기
    ```cs
    // 이벤트 종류는 다음과 같다.
    // Cancelled: Triggered when the opening of the object is cancelled text.
    // Copied: Triggered after the object is cloned.
    // Erased: Triggered when the object is flagged to be erased or is unerased.
    // Goodbye: Triggered when the object is about to be deleted from memory because its associated database is being destroyed.
    // Modified: Triggered when the object is modified.
    // ModifiedXData: Triggered when the XData attached to the object is modified.
    // ModifyUndone: Triggered when previous changes to the object are being undone.
    // ObjectClosed: Triggered when the object is closed.
    // OpenedForModify: Triggered before the object is modified.
    // Reappended: Triggered when the object is removed from the database after an Undo operation and then re-appended with a Redo operation.
    // SubObjectModified: Triggered when a subobject of the object is modified.
    // Unappended: Triggered when the object is removed from the database after an Undo operation. The following are some of the events used to respond to object changes at the Database level:
    // ObjectAppended: Triggered when an object is added to a database.
    // ObjectErased: Triggered when an object is erased or unerased from a database.
    // ObjectModified: Triggered when an object has been modified.
    // ObjectOpenedForModify: Triggered before an object is modified.
    // ObjectReappended: Triggered when an object is removed from a database after an Undo operation and then re-appended with a Redo operation.
    // ObjectUnappended: Triggered when an object is removed from a database after an Undo operation.
    
    // Global variable for polyline object
    Polyline acPoly = null;
 
    [CommandMethod("AddPlObjEvent")]
    public void AddPlObjEvent()
    {
        // Get the current document and database, and start a transaction
        Document acDoc = Application.DocumentManager.MdiActiveDocument;
        Database acCurDb = acDoc.Database;
 
        using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
        {
            // Open the Block table record for read
            BlockTable acBlkTbl;
            acBlkTbl = acTrans.GetObject(acCurDb.BlockTableId, OpenMode.ForRead) as BlockTable;
 
            // Open the Block table record Model space for write
            BlockTableRecord acBlkTblRec;
            acBlkTblRec = acTrans.GetObject(acBlkTbl[BlockTableRecord.ModelSpace], OpenMode.ForWrite) as BlockTableRecord;
 
            // Create a closed polyline
            acPoly = new Polyline();
            acPoly.AddVertexAt(0, new Point2d(1, 1), 0, 0, 0);
            acPoly.AddVertexAt(1, new Point2d(1, 2), 0, 0, 0);
            acPoly.AddVertexAt(2, new Point2d(2, 2), 0, 0, 0);
            acPoly.AddVertexAt(3, new Point2d(3, 3), 0, 0, 0);
            acPoly.AddVertexAt(4, new Point2d(3, 2), 0, 0, 0);
            acPoly.Closed = true;
 
            // Add the new object to the block table record and the transaction
            acBlkTblRec.AppendEntity(acPoly);
            acTrans.AddNewlyCreatedDBObject(acPoly, true);
 
            acPoly.Modified += new EventHandler(acPolyMod);
 
            // Save the new object to the database
            acTrans.Commit();
        }
    }
 
    [CommandMethod("RemovePlObjEvent")]
    public void RemovePlObjEvent()
    {
        if (acPoly != null)
        {
            // Get the current document and database, and start a transaction
            Document acDoc = Application.DocumentManager.MdiActiveDocument;
            Database acCurDb = acDoc.Database;
 
            using (Transaction acTrans = acCurDb.TransactionManager.StartTransaction())
            {
                // Open the polyline for read
                acPoly = acTrans.GetObject(acPoly.ObjectId, OpenMode.ForRead) as Polyline;
 
                if (acPoly.IsWriteEnabled == false)
                {
                    acTrans.GetObject(acPoly.ObjectId, OpenMode.ForWrite);
                }
 
                acPoly.Modified -= new EventHandler(acPolyMod);
                acPoly = null;
            }
        }
    }
 
    public void acPolyMod(object senderObj, EventArgs evtArgs)
    {
        Application.ShowAlertDialog("The area of " + acPoly.ToString() + " is: " + acPoly.Area);
    }
    ```
  - COM 기반 이벤트 등록하기
    ```cs
    // Global variable for AddCOMEvent and RemoveCOMEvent commands
    AcadApplication acAppCom;
 
    [CommandMethod("AddCOMEvent")]
    public void AddCOMEvent()
    {
        // Set the global variable to hold a reference to the application and
        // register the BeginFileDrop COM event
        acAppCom = Application.AcadApplication as AcadApplication;
        acAppCom.BeginFileDrop += new _DAcadApplicationEvents_BeginFileDropEventHandler(appComBeginFileDrop);
    }
 
    [CommandMethod("RemoveCOMEvent")]
    public void RemoveCOMEvent()
    {
        // Unregister the COM event handle
        acAppCom.BeginFileDrop -= new _DAcadApplicationEvents_BeginFileDropEventHandler(appComBeginFileDrop);
        acAppCom = null;
    }
 
    public void appComBeginFileDrop(string strFileName, ref bool bCancel)
    {
        // Display a message box prompting to continue inserting the DWG file
        if (System.Windows.Forms.MessageBox.Show("AutoCAD is about to load " + strFileName +
                                       "\nDo you want to continue loading this file?",
                                       "DWG File Dropped",
                                       System.Windows.Forms.MessageBoxButtons.YesNo) == 
                                       System.Windows.Forms.DialogResult.No)
        {
            bCancel = true;
        }
    }
    ```

#### 개발
  - .NET 애플리케이션 배포하기
    ```cs
    // APPLOAD 또는 NETLOAD를 통해 수동으로 로드하거나 자동으로 로드하는 방법이 있다.
    // 자동 로드에 대한 것은 다음을 참조할 것.
    // LOADCTRLS: Controls how and when the .NET assembly is loaded.
    //   1 - Load application upon detection of proxy object
    //   2 - Load the application at startup
    //   4 - Load the application at start of a command
    //   8 - Load the application at the request of a user or another application
    //   16 - Do not load the application
    //   32 - Load the application transparently
    // LOADER: Specifies which .NET assembly file to load.
    // MANAGED: Specifies the file that should be loaded is a .NET assembly or ObjectARX file. Set to 1 for .NET assembly files.
    
    using Microsoft.Win32;
    using System.Reflection;

    using Autodesk.AutoCAD.Runtime;
    using Autodesk.AutoCAD.ApplicationServices;
    using Autodesk.AutoCAD.DatabaseServices;

    [CommandMethod("RegisterMyApp")]
    public void RegisterMyApp()
    {
        // Get the AutoCAD Applications key
        string sProdKey = HostApplicationServices.Current.RegistryProductRootKey;
        string sAppName = "MyApp";

        RegistryKey regAcadProdKey = Registry.CurrentUser.OpenSubKey(sProdKey);
        RegistryKey regAcadAppKey = regAcadProdKey.OpenSubKey("Applications", true);

        // Check to see if the "MyApp" key exists
        string[] subKeys = regAcadAppKey.GetSubKeyNames();
        foreach (string subKey in subKeys)
        {
            // If the application is already registered, exit
            if (subKey.Equals(sAppName))
            {
                regAcadAppKey.Close();
                return;
            }
        }

        // Get the location of this module
        string sAssemblyPath = Assembly.GetExecutingAssembly().Location;

        // Register the application
        RegistryKey regAppAddInKey = regAcadAppKey.CreateSubKey(sAppName);
        regAppAddInKey.SetValue("DESCRIPTION", sAppName, RegistryValueKind.String);
        regAppAddInKey.SetValue("LOADCTRLS", 14, RegistryValueKind.DWord);
        regAppAddInKey.SetValue("LOADER", sAssemblyPath, RegistryValueKind.String);
        regAppAddInKey.SetValue("MANAGED", 1, RegistryValueKind.DWord);

        regAcadAppKey.Close();
    }

    [CommandMethod("UnregisterMyApp")]
    public void UnregisterMyApp()
    {
        // Get the AutoCAD Applications key
        string sProdKey = HostApplicationServices.Current.RegistryProductRootKey;
        string sAppName = "MyApp";

        RegistryKey regAcadProdKey = Registry.CurrentUser.OpenSubKey(sProdKey);
        RegistryKey regAcadAppKey = regAcadProdKey.OpenSubKey("Applications", true);

        // Delete the key for the application
        regAcadAppKey.DeleteSubKeyTree(sAppName);
        regAcadAppKey.Close();
    }
    ```
