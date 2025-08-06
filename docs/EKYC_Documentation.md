# EKYC System Documentation

## Overview

The Electronic Know Your Customer (EKYC) system provides identity verification functionality for users. The system supports two verification paths:

1. **Automatic Verification**: Uses SDK integration for automated ID card and face verification
2. **Manual Verification**: Fallback option when automatic verification fails, allowing users to manually submit their information and documents

### Tổng quan hệ thống

```mermaid
flowchart TD
    subgraph Components[Các thành phần chính của hệ thống EKYC]
        Store[EKYC Store<br>Zustand]
        API[API Integration]
        Camera[Camera Components]
        Form[Form Components]
        Tracking[Analytics Tracking]

        Store -->|State Management| Camera
        Store -->|State Management| Form
        Store -->|State Management| API

        Camera -->|Image Data| Store
        Form -->|User Data| Store
        Store -->|Submit Data| API
        API -->|Response Data| Store

        Tracking -->|Event Tracking| Camera
        Tracking -->|Event Tracking| Form
        Tracking -->|Event Tracking| API
    end
```

## System Architecture

### Key Components

1. **EKYC Store**: Central state management using Zustand
2. **API Integration**: Backend communication for verification and data submission
3. **Camera Components**: For capturing ID cards and selfies
4. **Form Components**: For manual data entry
5. **Tracking System**: Analytics for user journey through the verification process

### File Structure

```
src/screens/Menu/EKYC/
├── components/            # Reusable UI components
│   ├── camera/           # Camera capture components
│   ├── DropDownCustom.tsx # Custom dropdown component
│   ├── FooterComponent.tsx # Footer with action buttons
│   └── ImageSection.tsx   # Image display and capture UI
├── hooks/                 # Custom React hooks
│   ├── useEkycFlowActions.tsx # Actions for EKYC flow
│   ├── useManualEkycTracking.ts # Analytics tracking
│   └── useMaxAttemptController.tsx # Handles max attempts logic
├── store/                 # State management
│   ├── actions/           # Store actions
│   │   ├── ekycCheckEligibility.ts # Check if user can attempt verification
│   │   ├── ekycFaceCheck.ts # Face verification
│   │   ├── ekycIDCheck.ts # ID verification
│   │   ├── ekycManualSubmit.ts # Submit manual verification
│   │   └── getLatestManualEkyc.ts # Get latest manual verification
│   └── index.ts           # Store definition and types
├── utils/                 # Utility functions
│   └── imageUtils.ts      # Image processing utilities
├── EKYCScreen.tsx         # Main automatic verification screen
├── EKYCManualForm.tsx     # Manual verification form
├── EKYCManualReview.tsx   # Review screen for manual verification
└── SimpleCameraPage.tsx   # Camera capture page
```

## Data Flow

1. **User Initiates EKYC**:

   - Starts at `EKYCScreen.tsx` with introduction
   - Chooses ID type (CCCD or Passport)

2. **Automatic Verification**:

   - User captures ID front/back and face using SDK
   - System verifies images using `ekycIDCheck.ts` and `ekycFaceCheck.ts`
   - If successful, user is verified
   - If unsuccessful after maximum attempts, redirects to manual verification

3. **Manual Verification**:
   - User enters personal information in `EKYCManualForm.tsx`
   - Captures ID images and selfie using `SimpleCameraPage.tsx`
   - Reviews information in `EKYCManualReview.tsx`
   - Submits for manual review by admin using `ekycManualSubmit.ts`
   - User can check status and resubmit if rejected

### Biểu đồ luồng EKYC

```mermaid
flowchart TD
    Start[Người dùng bắt đầu EKYC] --> Intro[Màn hình giới thiệu]
    Intro --> IDChoice[Chọn loại giấy tờ]
    IDChoice --> AutoFlow[Luồng xác thực tự động]

    subgraph AutoFlow[Luồng xác thực tự động]
        FrontID[Chụp mặt trước CCCD/Passport] --> VerifyFront[Xác thực mặt trước]
        VerifyFront -->|Thành công| BackID[Chụp mặt sau CCCD]
        VerifyFront -->|Thất bại| RetryFront[Thử lại]
        RetryFront --> MaxAttempt{Đã đạt giới hạn?}
        MaxAttempt -->|Có| Manual
        MaxAttempt -->|Không| FrontID

        BackID --> VerifyBack[Xác thực mặt sau]
        VerifyBack -->|Thành công| FaceCapture[Chụp khuôn mặt]
        VerifyBack -->|Thất bại| RetryBack[Thử lại]
        RetryBack --> MaxAttemptBack{Đã đạt giới hạn?}
        MaxAttemptBack -->|Có| Manual
        MaxAttemptBack -->|Không| BackID

        FaceCapture --> VerifyFace[Xác thực khuôn mặt]
        VerifyFace -->|Thành công| Success[Xác thực thành công]
        VerifyFace -->|Thất bại| RetryFace[Thử lại]
        RetryFace --> MaxAttemptFace{Đã đạt giới hạn?}
        MaxAttemptFace -->|Có| Manual
        MaxAttemptFace -->|Không| FaceCapture
    end

    Manual[Chuyển sang xác thực thủ công] --> ManualFlow[Luồng xác thực thủ công]

    subgraph ManualFlow[Luồng xác thực thủ công]
        FormEntry[Nhập thông tin cá nhân] --> CaptureIDFront[Chụp mặt trước CCCD/Passport]
        CaptureIDFront --> CaptureIDBack[Chụp mặt sau CCCD]
        CaptureIDBack --> CaptureFace[Chụp khuôn mặt]
        CaptureFace --> Review[Xem lại thông tin]
        Review -->|Chỉnh sửa| FormEntry
        Review -->|Xác nhận| Submit[Gửi xác thực]
        Submit --> Pending[Chờ xét duyệt]
        Pending --> CheckStatus[Kiểm tra trạng thái]
        CheckStatus -->|Đã duyệt| Approved[Xác thực thành công]
        CheckStatus -->|Từ chối| Rejected[Xác thực thất bại]
        Rejected --> ViewReason[Xem lý do từ chối]
        ViewReason --> Resubmit[Gửi lại]
        Resubmit --> FormEntry
    end

    Success --> End[Hoàn thành EKYC]
    Approved --> End
```

## State Management

The EKYC system uses Zustand for state management with the following key states:

### Main States

```typescript
// From store/index.ts
export enum MANUAL_EKYC_STATE {
  CREATED = "CREATED",
  EDIT = "EDIT",
  CONFIRM = "CONFIRM",
  PENDING = "PENDING",
  APPROVED = "APPROVED",
  REJECT = "REJECT",
  SUBMITTED = "SUBMITTED",
  REVIEW = "REVIEW",
  TAKE_PHOTO = "TAKE_PHOTO",
  TAKE_FACE = "TAKE_FACE",
  TAKE_BACK_SIDE = "TAKE_BACK_SIDE",
  TAKE_FRONT_SIDE = "TAKE_FRONT_SIDE",
}

export interface IProfileVerificationState extends IBaseState {
  idType: EKYC_ID_TYPE;
  loading: boolean;
  currentStep: VERIFY_STEP;
  errorText: string;
  verificationInfo?: IdentityInformation;
  shouldRetakeFrontSide: boolean;
  verifyResult: boolean;
  ekycEligibility?: IEKYCEligibilityResponse;
  latestManualEkyc?: IEKYCManualLatestResponse;

  // Manual EKYC information
  manualEKYCInfo: {
    fullName: string;
    birthDate: string;
    sex: string;
    nation: string;
    rejectReason: string;
    state: MANUAL_EKYC_STATE;
  };

  // Manual EKYC images
  manualEKYCImage: {
    frontSide: string;
    backSide: string;
    face: string;
  };
}
```

### Key Actions

1. **setIDType**: Set the type of ID (CCCD or Passport)
2. **ekycIDCheck**: Verify ID card images
3. **ekycFaceCheck**: Verify face image
4. **uploadEkycImage**: Upload images to the server
5. **ekycManualSubmit**: Submit manual verification data
6. **getLatestManualEkyc**: Get the latest manual verification status
7. **setManualEKYCInfo**: Update manual verification information
8. **setManualEKYCImage**: Update manual verification images

## API Integration

### Key Endpoints

1. **ID Verification**: Verifies ID card images
2. **Face Verification**: Verifies face image and matches with ID
3. **Manual Submission**: Submits manual verification data
4. **Check Eligibility**: Checks if user can attempt verification
5. **Get Latest Manual EKYC**: Gets the latest manual verification status

### Image Handling

Images are handled using the utilities in `imageUtils.ts`:

1. **getEkycImageUrl**: Creates URL for EKYC images
2. **fetchEkycImageAsUri**: Fetches image and converts to local URI
3. **createEkycImageSource**: Creates source object for Image components

## User Flow

### Automatic Verification Flow

1. User starts at introduction screen
2. User selects ID type (CCCD or Passport)
3. User captures front ID using SDK
4. System verifies front ID
5. User captures back ID (if required)
6. System verifies back ID
7. User captures face
8. System verifies face and matches with ID
9. If all verifications pass, user is verified
10. If any verification fails after maximum attempts, user is redirected to manual verification

### Manual Verification Flow

1. User is redirected to manual form after automatic verification fails
2. User enters personal information (name, birth date, sex, nationality)
3. User captures ID front using camera
4. User captures ID back (if required)
5. User captures face
6. User reviews all information
7. User submits for manual review
8. User can check status and resubmit if rejected

### Biểu đồ trạng thái EKYC

```mermaid
stateDiagram-v2
    [*] --> INTRO: Bắt đầu EKYC

    state "Luồng tự động" as Auto {
        INTRO --> CHOOSE_TYPE_ID: Chọn loại giấy tờ
        CHOOSE_TYPE_ID --> FRONT_ID_CAPTURE: Chụp mặt trước
        FRONT_ID_CAPTURE --> FRONT_ID_VERIFY: Xác thực

        state FRONT_ID_VERIFY {
            [*] --> PROCESSING: Đang xử lý
            PROCESSING --> SUCCESS: Thành công
            PROCESSING --> FAILURE: Thất bại
            FAILURE --> [*]: Thử lại
        }

        FRONT_ID_VERIFY --> BACK_ID_CAPTURE: Thành công
        BACK_ID_CAPTURE --> BACK_ID_VERIFY: Xác thực

        state BACK_ID_VERIFY {
            [*] --> BACK_PROCESSING: Đang xử lý
            BACK_PROCESSING --> BACK_SUCCESS: Thành công
            BACK_PROCESSING --> BACK_FAILURE: Thất bại
            BACK_FAILURE --> [*]: Thử lại
        }

        BACK_ID_VERIFY --> FACE_CAPTURE: Thành công
        FACE_CAPTURE --> FACE_VERIFY: Xác thực

        state FACE_VERIFY {
            [*] --> FACE_PROCESSING: Đang xử lý
            FACE_PROCESSING --> FACE_SUCCESS: Thành công
            FACE_PROCESSING --> FACE_FAILURE: Thất bại
            FACE_FAILURE --> [*]: Thử lại
        }

        FACE_VERIFY --> CONFIRM_INFORMATION: Thành công
        CONFIRM_INFORMATION --> RESULT: Xác nhận
    }

    state "Luồng thủ công" as Manual {
        state MANUAL_EKYC_STATE {
            [*] --> CREATED: Khởi tạo form
            CREATED --> TAKE_FRONT_SIDE: Chụp mặt trước
            TAKE_FRONT_SIDE --> TAKE_BACK_SIDE: Chụp mặt sau
            TAKE_BACK_SIDE --> TAKE_FACE: Chụp khuôn mặt
            TAKE_FACE --> REVIEW: Xem lại
            REVIEW --> EDIT: Chỉnh sửa
            EDIT --> REVIEW: Quay lại xem lại
            REVIEW --> CONFIRM: Xác nhận
            CONFIRM --> SUBMITTED: Gửi xét duyệt
            SUBMITTED --> PENDING: Chờ duyệt
            PENDING --> APPROVED: Được duyệt
            PENDING --> REJECT: Từ chối
            REJECT --> EDIT: Chỉnh sửa và gửi lại
        }
    }

    state MAX_ATTEMPTS <<fork>>
    FRONT_ID_VERIFY --> MAX_ATTEMPTS: Vượt quá số lần thử
    BACK_ID_VERIFY --> MAX_ATTEMPTS: Vượt quá số lần thử
    FACE_VERIFY --> MAX_ATTEMPTS: Vượt quá số lần thử

    MAX_ATTEMPTS --> MANUAL_EKYC_STATE: Chuyển sang xác thực thủ công

    RESULT --> [*]: Hoàn thành
    APPROVED --> [*]: Hoàn thành
```

## Analytics Tracking

The system uses `useManualEkycTracking.ts` to track user journey:

1. **MANUAL_EKYC_STARTED**: When manual EKYC flow starts
2. **MANUAL_EKYC_INFO_LOADED**: When form is loaded
3. **MANUAL_EKYC_CAMERA_OPENED**: When camera is opened
4. **MANUAL_EKYC_PHOTO_CONFIRMED**: When photo is confirmed
5. **MANUAL_EKYC_PREVIEW_VIEWED**: When preview screen is viewed
6. **MANUAL_EKYC_EDIT_MODE_OPENED**: When edit mode is opened
7. **MANUAL_EKYC_SUBMITTED**: When form is submitted
8. **MANUAL_EKYC_INSTRUCTION_OPENED**: When instructions are opened
9. **MANUAL_EKYC_CANCELED**: When user cancels the flow

## Error Handling

1. **Maximum Attempts**: If user exceeds maximum verification attempts, they are redirected to manual verification
2. **API Errors**: Displayed to user with option to retry
3. **Image Capture Errors**: User can retake photos

## Implementation Details

### Manual EKYC Form (`EKYCManualForm.tsx`)

This component handles the manual data entry form with the following features:

- Form validation using React Hook Form
- Date picker for birth date
- Country selection dropdown
- Image capture integration
- Form state persistence

### Manual EKYC Review (`EKYCManualReview.tsx`)

This component displays the review screen with:

- All entered information for confirmation
- Image previews
- Edit functionality
- Submit button

### Simple Camera Page (`SimpleCameraPage.tsx`)

This component handles image capture with:

- Camera integration
- Image preview
- Retake functionality
- Image size tracking

### Maximum Attempt Controller (`useMaxAttemptController.tsx`)

This hook manages the maximum verification attempts:

- Tracks number of attempts
- Shows alert when maximum attempts reached
- Redirects to manual verification

### Biểu đồ tuần tự EKYC

```mermaid
sequenceDiagram
    participant User as Người dùng
    participant UI as UI Components
    participant Store as EKYC Store
    participant SDK as EKYC SDK
    participant API as Backend API
    participant Admin as Admin System

    User->>UI: Bắt đầu EKYC
    UI->>Store: Khởi tạo state

    %% Automatic Flow
    User->>UI: Chọn loại ID (CCCD/Passport)
    UI->>Store: setIDType()
    User->>SDK: Chụp mặt trước ID
    SDK->>Store: Lưu ảnh
    Store->>API: ekycIDCheck()
    API-->>Store: Kết quả xác thực

    alt Xác thực mặt trước thành công
        User->>SDK: Chụp mặt sau ID
        SDK->>Store: Lưu ảnh
        Store->>API: ekycIDCheck()
        API-->>Store: Kết quả xác thực

        alt Xác thực mặt sau thành công
            User->>SDK: Chụp khuôn mặt
            SDK->>Store: Lưu ảnh
            Store->>API: ekycFaceCheck()
            API-->>Store: Kết quả xác thực khuôn mặt

            alt Xác thực khuôn mặt thành công
                Store->>UI: Hiển thị thành công
                UI->>User: Thông báo xác thực thành công
            else Xác thực khuôn mặt thất bại
                Store->>UI: Hiển thị lỗi
                UI->>User: Thông báo thử lại
            end
        else Xác thực mặt sau thất bại
            Store->>UI: Hiển thị lỗi
            UI->>User: Thông báo thử lại
        end
    else Xác thực mặt trước thất bại
        Store->>UI: Hiển thị lỗi
        UI->>User: Thông báo thử lại
    end

    %% Manual Flow after maximum attempts
    alt Vượt quá số lần thử
        Store->>UI: Chuyển sang xác thực thủ công
        UI->>User: Hiển thị form xác thực thủ công
        User->>UI: Nhập thông tin cá nhân
        UI->>Store: setManualEKYCInfo()
        User->>UI: Chụp ảnh ID và khuôn mặt
        UI->>Store: setManualEKYCImage()
        User->>UI: Xem lại và xác nhận
        UI->>Store: ekycManualSubmit()
        Store->>API: Gửi thông tin xác thực thủ công
        API-->>Store: Xác nhận đã nhận
        Store->>UI: Hiển thị trạng thái chờ duyệt
        UI->>User: Thông báo đã gửi, chờ duyệt

        %% Admin review process
        API->>Admin: Gửi yêu cầu xét duyệt
        Admin-->>API: Kết quả xét duyệt

        %% User checks status later
        User->>UI: Kiểm tra trạng thái
        UI->>Store: getLatestManualEkyc()
        Store->>API: Lấy trạng thái mới nhất
        API-->>Store: Trạng thái xét duyệt
        Store->>UI: Cập nhật UI
        UI->>User: Hiển thị kết quả xét duyệt
    end
```

## Integration with External Systems

### EKYC SDK Integration

The system integrates with an external EKYC SDK for automatic verification:

- Configuration in `useEkycFlowActions.tsx`
- SDK initialization and usage
- Result handling and parsing

### Backend API Integration

The system communicates with backend APIs for:

- Image verification
- Data submission
- Status checking

## Testing

To test the EKYC system:

1. **Automatic Verification**:

   - Test with valid ID cards
   - Test with invalid ID cards
   - Test with different lighting conditions
   - Test maximum attempts logic

2. **Manual Verification**:
   - Test form validation
   - Test image capture
   - Test submission process
   - Test rejection and resubmission flow

## Common Issues and Solutions

1. **Image Quality Issues**:

   - Ensure good lighting
   - Hold camera steady
   - Follow on-screen guidelines

2. **Verification Failures**:

   - Check ID card is valid and not expired
   - Ensure face is clearly visible
   - Try manual verification if automatic fails

3. **Form Submission Errors**:
   - Check all required fields are filled
   - Verify image uploads are complete
   - Check network connection

## Future Improvements

1. **Offline Support**: Add capability to capture and store verification data offline
2. **Enhanced Image Processing**: Improve image quality checks before submission
3. **Multi-language Support**: Extend support for more languages
4. **Accessibility Improvements**: Enhance for users with disabilities

## Cấu trúc lớp

```mermaid
classDiagram
    class EKYCStore {
        +idType: EKYC_ID_TYPE
        +loading: boolean
        +currentStep: VERIFY_STEP
        +errorText: string
        +verificationInfo: IdentityInformation
        +manualEKYCInfo: object
        +manualEKYCImage: object
        +setIDType()
        +ekycIDCheck()
        +ekycFaceCheck()
        +uploadEkycImage()
        +setManualEKYCInfo()
        +setManualEKYCImage()
        +ekycManualSubmit()
        +getLatestManualEkyc()
    }

    class EKYCScreen {
        +handleBack()
        +onBackToHome()
        +onTryAgain()
        +onPressButton()
        +renderContent()
        +renderFaceLivenessResult()
    }

    class EKYCManualForm {
        +form: FormData
        +handleSubmit()
        +handleImageCapture()
        +navigateToCamera()
        +renderFormFields()
        +renderImageSection()
    }

    class EKYCManualReview {
        +handleEdit()
        +handleSubmit()
        +renderFormFields()
        +renderImageSection()
    }

    class SimpleCameraPage {
        +onTakePicture()
        +handleShowInstructions()
        +renderCamera()
    }

    class useManualEkycTracking {
        +trackManualEkycStarted()
        +trackManualEkycInfoLoaded()
        +trackManualEkycCameraOpened()
        +trackManualEkycPhotoConfirmed()
        +trackManualEkycPreviewViewed()
        +trackManualEkycSubmitted()
    }

    class useMaxAttemptController {
        +onManualEKYCForm()
        +onRedirectToManualEKYCForm()
    }

    class imageUtils {
        +getEkycImageUrl()
        +fetchEkycImageAsUri()
        +createEkycImageSource()
    }

    EKYCStore <-- EKYCScreen: uses
    EKYCStore <-- EKYCManualForm: uses
    EKYCStore <-- EKYCManualReview: uses
    EKYCStore <-- SimpleCameraPage: uses

    EKYCScreen --> EKYCManualForm: navigates to
    EKYCManualForm --> SimpleCameraPage: navigates to
    EKYCManualForm --> EKYCManualReview: navigates to

    EKYCScreen --> useMaxAttemptController: uses
    EKYCManualForm --> useManualEkycTracking: uses
    EKYCManualReview --> useManualEkycTracking: uses
    SimpleCameraPage --> useManualEkycTracking: uses

    EKYCManualReview --> imageUtils: uses
    SimpleCameraPage --> imageUtils: uses
```

## Conclusion

The EKYC system provides a robust verification solution with both automatic and manual paths. The manual verification flow serves as a critical fallback when automatic verification fails, ensuring all users can complete the verification process.
