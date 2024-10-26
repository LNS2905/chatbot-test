<template>
  <div class="chat-container">
    <h2 class="chat-header">NoLe ChatBot của Ticketo</h2>

    <div class="messages-container" ref="messagesContainer">
      <div
        v-for="(message, index) in messages"
        :key="index"
        :class="[
          'message',
          message.role === 'user' ? 'user-message' : 'bot-message',
        ]">
        <div class="message-content">
          <div
            v-for="(part, partIndex) in message.parts"
            :key="partIndex"
            :class="part.type === 'html' ? 'html-part' : 'text-part'">
            <span
              v-if="part.type === 'text'"
              v-html="formatMessage(part.content)"></span>
            <span v-else v-html="part.content"></span>
          </div>
        </div>
        <div class="message-time">{{ formatTime(message.timestamp) }}</div>
      </div>

      <div v-if="isLoading" class="loading-message">
        <div class="typing-indicator">
          <span></span><span></span><span></span>
        </div>
      </div>
    </div>

    <div class="input-container">
      <textarea
        v-model="userInput"
        @keyup.enter.exact="sendMessage"
        placeholder="Nhập tin nhắn của bạn..."
        :disabled="isLoading"
        rows="3"></textarea>
      <button @click="sendMessage" :disabled="isLoading || !userInput.trim()">
        Gửi
      </button>
    </div>
  </div>
</template>

<script>
import { GoogleGenerativeAI } from "@google/generative-ai";
import axios from "axios";
import DOMPurify from "dompurify";
import { marked } from "marked";

export default {
  name: "GeminiChatbot",
  data() {
    return {
      messages: [],
      userInput: "",
      isLoading: false,
      model: null,
      chat: null,
      systemPrompt: `Bạn là một trợ lý AI chuyên nghiệp, làm việc cho website bán vé xem phim Ticketo.
Hãy phân tích ý định của người dùng từ tin nhắn sau và trả về kết quả theo format [ý định], [thông tin bổ sung]:

Tin nhắn của người dùng: {userMessage}

Các ý định có thể có:
- tim_phim: Người dùng muốn tìm kiếm thông tin về một bộ phim cụ thể. Thông tin bổ sung: Tên phim.
- goi_y_phim: Người dùng muốn được gợi ý phim.
- hoi_dap: Người dùng muốn hỏi đáp về các vấn đề khác (ví dụ: chính sách, thanh toán, etc.).

Ví dụ:
- Tin nhắn của người dùng: "Bạn có thể cho tôi biết thông tin về phim Lật Mặt 6?" -> Kết quả: tim_phim, Lật Mặt 6
- Tin nhắn của người dùng: "Bạn có gợi ý phim nào hay không?" -> Kết quả: goi_y_phim,
- Tin nhắn của người dùng: "Làm thế nào để đặt vé online?" -> Kết quả: hoi_dap,
`,
      tmdbApiKey: "28859d876d66d7d344a5e1652b06949c",
    };
  },
  async created() {
    await this.initializeChat();
  },
  methods: {
    async initializeChat() {
      try {
        const genAI = new GoogleGenerativeAI(
          import.meta.env.VITE_GEMINI_API_KEY
        );
        this.model = genAI.getGenerativeModel({ model: "gemini-1.5-pro" });

        // Khởi tạo chat với system prompt
        this.chat = this.model.startChat({
          history: [
            {
              role: "user",
              parts: [{ text: this.systemPrompt }],
            },
          ],
          generationConfig: {
            maxOutputTokens: 5000,
            temperature: 0.5,
            topP: 0.1,
            topK: 16,
          },
        });
      } catch (error) {
        console.error("Error initializing chat:", error);
      }
    },

    async getMovieInfo(movieTitle) {
      try {
        const response = await axios.get(
          `https://api.themoviedb.org/3/search/movie?api_key=${this.tmdbApiKey}&query=${movieTitle}&language=vi-VN`
        );

        if (response.data.results.length > 0) {
          const movie = response.data.results[0];
          const movieInfo = {
            title: movie.title,
            overview: movie.overview,
            poster: `https://image.tmdb.org/t/p/w500${movie.poster_path}`,
            releaseDate: movie.release_date,
            voteAverage: movie.vote_average,
          };
          return movieInfo;
        } else {
          return null;
        }
      } catch (error) {
        console.error("Error getting movie info:", error);
        return null;
      }
    },

    async getMovieSuggestions() {
      try {
        const response = await axios.get(
          `https://api.themoviedb.org/3/movie/popular?api_key=${this.tmdbApiKey}&language=vi-VN`
        );

        if (response.data.results.length > 0) {
          const movies = response.data.results.slice(0, 3); // Lấy 3 phim đầu
          return movies.map((movie) => ({
            title: movie.title,
            overview: movie.overview,
            poster: `https://image.tmdb.org/t/p/w500${movie.poster_path}`,
            releaseDate: movie.release_date,
            voteAverage: movie.vote_average,
          }));
        } else {
          return null;
        }
      } catch (error) {
        console.error("Error getting movie suggestions:", error);
        return null;
      }
    },

    async sendMessage(event) {
      if (event?.shiftKey) return;
      if (!this.userInput.trim() || this.isLoading) return;
      if (!this.chat) {
        await this.initializeChat();
        if (!this.chat) {
          console.error("Failed to initialize chat");
          return;
        }
      }

      const userMessage = this.userInput.trim();
      this.userInput = "";
      this.isLoading = true;

      // Add user message
      this.addMessage({
        role: "user",
        parts: [{ type: "text", content: userMessage }],
        timestamp: new Date(),
      });

      try {
        // Gọi API của Gemini để phân tích ý định
        const analysisPrompt = this.systemPrompt.replace(
          "{userMessage}",
          userMessage
        );
        const analysisResult = await this.chat.sendMessage(analysisPrompt);
        const analysisResponse = await analysisResult.response;
        const [intent, additionalInfo] = analysisResponse.text().split(", ");
        console.log(
          "Phản hồi từ Gemini:",
          analysisResponse.candidates[0].content.parts[0].text
        );

        if (intent === "[tim_phim]") {
          const movieInfo = await this.getMovieInfo(additionalInfo);
          if (movieInfo) {
            this.addMessage({
              role: "model",
              parts: [
                { type: "text", content: `**${movieInfo.title}**` },
                {
                  type: "text",
                  content: `*Mô tả:* ${movieInfo.overview}`,
                },
                {
                  type: "text",
                  content: `*Ngày phát hành:* ${movieInfo.releaseDate}`,
                },
                {
                  type: "text",
                  content: `*Đánh giá:* ${movieInfo.voteAverage}/10`,
                },
                {
                  type: "html",
                  content: `<img src="${movieInfo.poster}" alt="${movieInfo.title} poster" style="max-width: 300px;">`,
                },
              ],
              timestamp: new Date(),
            });
          } else {
            this.addMessage({
              role: "model",
              parts: [
                {
                  type: "text",
                  content: `Xin lỗi, tôi không tìm thấy thông tin về phim "${additionalInfo}".`,
                },
              ],
              timestamp: new Date(),
            });
          }
        } else if (intent === "[goi_y_phim]") {
          const movieSuggestions = await this.getMovieSuggestions();

          if (movieSuggestions) {
            // Tạo prompt cho Gemini
            let prompt = "Dưới đây là một số phim hay bạn có thể thích:\n\n";
            movieSuggestions.forEach((movie) => {
              prompt += `**${movie.title}** (*${
                movie.releaseDate
              }*) - ${movie.overview.slice(0, 100)}...\n`; //Giới thiệu ngắn gọn
            });

            const result = await this.chat.sendMessage(prompt);
            const response = await result.response;

            this.addMessage({
              role: "model",
              parts: [{ type: "text", content: response.text() }],
              timestamp: new Date(),
            });
          } else {
            this.addMessage({
              role: "model",
              parts: [
                {
                  type: "text",
                  content: "Xin lỗi, tôi không tìm thấy phim nào.",
                },
              ],
              timestamp: new Date(),
            });
          }
        } else if (intent === "[hoi_dap]") {
          // Xử lý câu hỏi thông thường
          const result = await this.chat.sendMessage(userMessage);
          const response = await result.response;
          this.addMessage({
            role: "model",
            parts: [{ type: "text", content: response.text() }],
            timestamp: new Date(),
          });
        } else {
          // Xử lý trường hợp Gemini không thể phân tích ý định
          this.addMessage({
            role: "model",
            parts: [
              {
                type: "text",
                content:
                  "Xin lỗi, tôi không hiểu ý của bạn. Vui lòng diễn đạt lại.",
              },
            ],
            timestamp: new Date(),
          });
        }
      } catch (error) {
        console.error("Error:", error);
        this.addMessage({
          role: "model",
          parts: [
            {
              type: "text",
              content: "Xin lỗi, đã có lỗi xảy ra. Vui lòng thử lại sau.",
            },
          ],
          timestamp: new Date(),
        });

        // Thử khởi tạo lại chat nếu có lỗi
        await this.initializeChat();
      } finally {
        this.isLoading = false;
      }
    },

    addMessage(message) {
      this.messages.push(message);
      this.$nextTick(() => {
        this.scrollToBottom();
      });
    },

    scrollToBottom() {
      const container = this.$refs.messagesContainer;
      if (container) {
        container.scrollTop = container.scrollHeight;
      }
    },

    formatMessage(text) {
      try {
        // Xử lý các trường hợp đặc biệt
        text = text
          .replace(/\*\*(.*?)\*\*/g, "<strong>$1</strong>") // Bold text
          .replace(/\*(.*?)\*/g, "<em>$1</em>") // Italic text
          .replace(/`(.*?)`/g, "<code>$1</code>"); // Inline code

        return DOMPurify.sanitize(marked(text), {
          ALLOWED_TAGS: [
            "p",
            "br",
            "strong",
            "em",
            "code",
            "pre",
            "ul",
            "ol",
            "li",
            "blockquote",
            "a",
            "img",
          ],
          ALLOWED_ATTR: ["href", "src", "alt"],
        });
      } catch (error) {
        console.error("Error formatting message:", error);
        return text; // Trả về text gốc nếu có lỗi
      }
    },

    formatTime(date) {
      return new Date(date).toLocaleTimeString("vi-VN", {
        hour: "2-digit",
        minute: "2-digit",
      });
    },
  },
};
</script>

<style scoped>
.chat-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  height: 90vh;
  display: flex;
  flex-direction: column;
  background-color: #f5f5f5;
  border-radius: 10px;
  box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);
}

.chat-header {
  text-align: center;
  margin-bottom: 20px;
  color: #333;
}

.messages-container {
  flex: 1;
  overflow-y: auto;
  padding: 10px;
  background-color: white;
  border-radius: 8px;
  margin-bottom: 20px;
}

.message {
  margin: 10px 0;
  padding: 10px 15px;
  border-radius: 15px;
  max-width: 85%;
  word-wrap: break-word;
}

.user-message {
  background-color: #007bff;
  color: white;
  margin-left: auto;
}

.bot-message {
  background-color: #e9ecef;
  color: #333;
  margin-right: auto;
}

.message-time {
  font-size: 0.8em;
  margin-top: 5px;
  opacity: 0.7;
}

.input-container {
  display: flex;
  gap: 10px;
}

textarea {
  flex: 1;
  padding: 12px;
  border: 1px solid #ddd;
  border-radius: 5px;
  outline: none;
  resize: none;
  font-family: inherit;
}

button {
  padding: 12px 24px;
  background-color: #007bff;
  color: white;
  border: none;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
  height: fit-content;
}

button:disabled {
  background-color: #cccccc;
  cursor: not-allowed;
}

button:hover:not(:disabled) {
  background-color: #0056b3;
}

.loading-message {
  display: flex;
  justify-content: flex-start;
  margin: 10px 0;
}

.typing-indicator {
  background-color: #e9ecef;
  padding: 15px 20px;
  border-radius: 15px;
  display: flex;
  gap: 5px;
}

.typing-indicator span {
  width: 8px;
  height: 8px;
  background-color: #666;
  border-radius: 50%;
  animation: typing 1s infinite ease-in-out;
}

.typing-indicator span:nth-child(2) {
  animation-delay: 0.2s;
}

.typing-indicator span:nth-child(3) {
  animation-delay: 0.4s;
}

@keyframes typing {
  0%,
  100% {
    transform: translateY(0);
  }
  50% {
    transform: translateY(-5px);
  }
}

.message-content :deep(p) {
  margin: 0.5em 0;
}

.message-content :deep(ul),
.message-content :deep(ol) {
  margin: 0.5em 0;
  padding-left: 1.5em;
}

.message-content :deep(li) {
  margin: 0.3em 0;
}

.message-content :deep(strong) {
  font-weight: bold;
}

.message-content :deep(em) {
  font-style: italic;
}

.message-content :deep(code) {
  background-color: rgba(0, 0, 0, 0.1);
  padding: 0.2em 0.4em;
  border-radius: 3px;
  font-family: monospace;
}

.message-content :deep(pre) {
  background-color: #f8f9fa;
  padding: 1em;
  border-radius: 5px;
  overflow-x: auto;
}

.message-content :deep(blockquote) {
  border-left: 4px solid #ddd;
  margin: 0.5em 0;
  padding-left: 1em;
  color: #666;
}

.message-content :deep(a) {
  color: #007bff;
  text-decoration: none;
}

.message-content :deep(a:hover) {
  text-decoration: underline;
}

.message-content :deep(img) {
  max-width: 100%;
  height: auto;
  border-radius: 5px;
}

/* Đặc biệt cho tin nhắn của user */
.user-message .message-content :deep(a),
.user-message .message-content :deep(code) {
  color: white;
}

.user-message .message-content :deep(pre) {
  background-color: rgba(255, 255, 255, 0.1);
}

.user-message .message-content :deep(blockquote) {
  border-left-color: rgba(255, 255, 255, 0.5);
  color: rgba(255, 255, 255, 0.8);
}
</style>
