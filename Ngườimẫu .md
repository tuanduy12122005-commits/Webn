import { useState, useRef } from 'react';
import { GoogleGenAI } from '@google/genai';
import { Upload, Image as ImageIcon, Sparkles, Loader2, RefreshCw } from 'lucide-react';

// Khởi tạo Gemini API với API Key của bạn
const ai = new GoogleGenAI({ apiKey: process.env.GEMINI_API_KEY });

export default function App() {
  // State lưu trữ ảnh người mẫu
  const [modelImage, setModelImage] = useState<File | null>(null);
  const [modelPreview, setModelPreview] = useState<string | null>(null);
  
  // State lưu trữ ảnh trang phục
  const [garmentImage, setGarmentImage] = useState<File | null>(null);
  const [garmentPreview, setGarmentPreview] = useState<string | null>(null);
  
  // State lưu trữ kết quả và trạng thái
  const [resultImage, setResultImage] = useState<string | null>(null);
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const modelInputRef = useRef<HTMLInputElement>(null);
  const garmentInputRef = useRef<HTMLInputElement>(null);

  // Hàm xử lý khi người dùng chọn ảnh
  const handleImageUpload = (
    e: React.ChangeEvent<HTMLInputElement>,
    setImage: (file: File | null) => void,
    setPreview: (url: string | null) => void
  ) => {
    const file = e.target.files?.[0];
    if (file) {
      setImage(file);
      const reader = new FileReader();
      reader.onloadend = () => {
        setPreview(reader.result as string);
      };
      reader.readAsDataURL(file);
    }
  };

  // Hàm chuyển đổi File ảnh sang định dạng Base64 để gửi cho AI
  const fileToBase64 = (file: File): Promise<{ mimeType: string; data: string }> => {
    return new Promise((resolve, reject) => {
      const reader = new FileReader();
      reader.readAsDataURL(file);
      reader.onload = () => {
        const result = reader.result as string;
        const mimeType = result.split(';')[0].split(':')[1];
        const data = result.split(',')[1];
        resolve({ mimeType, data });
      };
      reader.onerror = (error) => reject(error);
    });
  };

  // Hàm gọi AI để tạo ảnh ghép
  const handleGenerate = async () => {
    if (!modelImage || !garmentImage) return;

    setIsLoading(true);
    setError(null);
    setResultImage(null);

    try {
      const modelData = await fileToBase64(modelImage);
      const garmentData = await fileToBase64(garmentImage);

      // Gọi Gemini API (Model: gemini-2.5-flash-image)
      const response = await ai.models.generateContent({
        model: 'gemini-2.5-flash-image',
        contents: {
          parts: [
            {
              inlineData: {
                data: modelData.data,
                mimeType: modelData.mimeType,
              },
            },
            {
              inlineData: {
                data: garmentData.data,
                mimeType: garmentData.mimeType,
              },
            },
            {
              // Câu lệnh (Prompt) yêu cầu AI ghép đồ
              text: 'A photorealistic image of the person in the first image wearing the exact clothing item shown in the second image. The clothing should fit perfectly and naturally on the person\'s body. Keep the original person\'s face, hair, pose, and the original background intact.',
            },
          ],
        },
      });

      // Trích xuất ảnh kết quả từ phản hồi của AI
      let generatedImageUrl = null;
      if (response.candidates && response.candidates[0].content.parts) {
        for (const part of response.candidates[0].content.parts) {
          if (part.inlineData) {
            generatedImageUrl = `data:${part.inlineData.mimeType || 'image/png'};base64,${part.inlineData.data}`;
            break;
          }
        }
      }

      if (generatedImageUrl) {
        setResultImage(generatedImageUrl);
      } else {
        throw new Error('Không tìm thấy ảnh kết quả từ AI.');
      }
    } catch (err: any) {
      console.error('Generation error:', err);
      setError(err.message || 'Đã xảy ra lỗi trong quá trình tạo ảnh. Vui lòng thử lại.');
    } finally {
      setIsLoading(false);
    }
  };

  // Hàm reset lại toàn bộ ứng dụng
  const resetAll = () => {
    setModelImage(null);
    setModelPreview(null);
    setGarmentImage(null);
    setGarmentPreview(null);
    setResultImage(null);
    setError(null);
    if (modelInputRef.current) modelInputRef.current.value = '';
    if (garmentInputRef.current) garmentInputRef.current.value = '';
  };

  return (
    <div className="min-h-screen bg-zinc-50 text-zinc-900 font-sans pb-20">
      {/* Header */}
      <header className="bg-white border-b border-zinc-200 sticky top-0 z-10">
        <div className="max-w-6xl mx-auto px-6 h-16 flex items-center justify-between">
          <div className="flex items-center gap-2">
            <div className="w-8 h-8 bg-indigo-600 rounded-lg flex items-center justify-center">
              <Sparkles className="w-5 h-5 text-white" />
            </div>
            <h1 className="text-xl font-semibold tracking-tight">AI Virtual Try-On</h1>
          </div>
          <button onClick={resetAll} className="text-sm font-medium text-zinc-500 hover:text-zinc-900 flex items-center gap-1.5">
            <RefreshCw className="w-4 h-4" /> Làm mới
          </button>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-6 mt-10 space-y-10">
        <div className="text-center max-w-2xl mx-auto space-y-3">
          <h2 className="text-4xl font-bold tracking-tight text-zinc-900">Thử trang phục bằng AI</h2>
          <p className="text-lg text-zinc-500">
            Tải lên ảnh người mẫu và ảnh trang phục bạn muốn thử. AI sẽ tự động ghép trang phục lên người mẫu một cách chân thực nhất.
          </p>
        </div>

        {/* Khu vực Upload Ảnh */}
        <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
          {/* Upload Người mẫu */}
          <div className="space-y-3">
            <label className="block text-sm font-semibold text-zinc-900">1. Ảnh người mẫu</label>
            <div 
              onClick={() => modelInputRef.current?.click()}
              className={`relative aspect-[3/4] rounded-2xl border-2 border-dashed transition-all cursor-pointer overflow-hidden group ${modelPreview ? 'border-transparent bg-zinc-100' : 'border-zinc-300 hover:border-indigo-400 bg-white'}`}
            >
              {modelPreview ? (
                <img src={modelPreview} alt="Model preview" className="w-full h-full object-cover" />
              ) : (
                <div className="absolute inset-0 flex flex-col items-center justify-center text-zinc-500 p-6 text-center">
                  <ImageIcon className="w-6 h-6 text-zinc-400 mb-2" />
                  <p className="font-medium text-zinc-900 mb-1">Nhấn để tải lên</p>
                </div>
              )}
              <input ref={modelInputRef} type="file" accept="image/*" className="hidden" onChange={(e) => handleImageUpload(e, setModelImage, setModelPreview)} />
            </div>
          </div>

          {/* Upload Trang phục */}
          <div className="space-y-3">
            <label className="block text-sm font-semibold text-zinc-900">2. Ảnh trang phục</label>
            <div 
              onClick={() => garmentInputRef.current?.click()}
              className={`relative aspect-[3/4] rounded-2xl border-2 border-dashed transition-all cursor-pointer overflow-hidden group ${garmentPreview ? 'border-transparent bg-zinc-100' : 'border-zinc-300 hover:border-indigo-400 bg-white'}`}
            >
              {garmentPreview ? (
                <img src={garmentPreview} alt="Garment preview" className="w-full h-full object-cover" />
              ) : (
                <div className="absolute inset-0 flex flex-col items-center justify-center text-zinc-500 p-6 text-center">
                  <ImageIcon className="w-6 h-6 text-zinc-400 mb-2" />
                  <p className="font-medium text-zinc-900 mb-1">Nhấn để tải lên</p>
                </div>
              )}
              <input ref={garmentInputRef} type="file" accept="image/*" className="hidden" onChange={(e) => handleImageUpload(e, setGarmentImage, setGarmentPreview)} />
            </div>
          </div>
        </div>

        {/* Nút Tạo Ảnh */}
        <div className="flex flex-col items-center justify-center pt-4">
          <button
            onClick={handleGenerate}
            disabled={!modelImage || !garmentImage || isLoading}
            className={`flex items-center justify-center gap-2 px-8 py-4 rounded-full font-semibold text-lg transition-all ${(!modelImage || !garmentImage || isLoading) ? 'bg-zinc-200 text-zinc-400 cursor-not-allowed' : 'bg-indigo-600 text-white hover:bg-indigo-700'}`}
          >
            {isLoading ? <><Loader2 className="w-5 h-5 animate-spin" /> Đang xử lý bằng AI...</> : <><Sparkles className="w-5 h-5" /> Tạo ảnh thử đồ</>}
          </button>
          {error && <div className="mt-4 p-4 bg-red-50 text-red-600 rounded-xl text-sm max-w-md text-center">{error}</div>}
        </div>

        {/* Khu vực Hiển thị Kết quả */}
        {resultImage && (
          <div className="pt-10 border-t border-zinc-200">
            <div className="text-center mb-8">
              <h3 className="text-2xl font-bold text-zinc-900">Kết quả</h3>
            </div>
            <div className="max-w-2xl mx-auto">
              <div className="bg-white p-4 rounded-3xl shadow-xl border border-zinc-100">
                <div className="relative aspect-[3/4] rounded-2xl overflow-hidden bg-zinc-100">
                  <img src={resultImage} alt="Result" className="w-full h-full object-cover" />
                </div>
                <div className="mt-4 flex justify-end">
                  <a href={resultImage} download="virtual-try-on-result.png" className="px-6 py-2.5 bg-zinc-900 text-white text-sm font-medium rounded-xl hover:bg-zinc-800">
                    Tải ảnh xuống
                  </a>
                </div>
              </div>
            </div>
          </div>
        )}
      </main>
    </div>
  );
}
