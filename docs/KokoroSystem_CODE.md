```python

import numpy as np
from typing import Dict, List, Tuple, Optional, Callable, Any
import math
from dataclasses import dataclass, field
from enum import Enum
import time
import random

class PMCStatus(Enum):
    COHERENT = "COHERENT"
    AT_RISK = "AT_RISK"
    VIOLATED = "VIOLATED"

class EmotionLayer(Enum):
    SURFACE = 1    # 驚きなど
    MID = 2        # 喜び、悲しみなど  
    CORE = 3       # 罪悪感、誇りなど

class TimeOrientation(Enum):
    PAST = 1
    PRESENT = 2
    FUTURE = 3

class VectorDirection(Enum):
    SELF = 1
    OTHER = 2
    BIDIRECTIONAL = 3

@dataclass
class EmotionStructure:
    """Emotion Structure Theory の4軸表現 + 動的相互作用"""
    integrity: float  # C: Consistency (0.0-1.0)
    layer: EmotionLayer  # L: Layer
    time: TimeOrientation  # T: Time
    vector: VectorDirection  # V: Vector
    
    def compute_emotion_value(self) -> float:
        """E = C × L × T × V による感情値計算"""
        return self.integrity * self.layer.value * self.time.value * self.vector.value
    
    def to_vector(self) -> List[float]:
        return [
            self.integrity,
            self.layer.value / 3.0,
            self.time.value / 3.0,
            self.vector.value / 3.0
        ]

@dataclass
class KokoroResonanceVector:
    """Kokoro共振ベクトル（動的状態管理付き）"""
    ER: float = 0.0  # Emotional Resonance (0.0-3.0)
    GR: float = 0.0  # Goal Resonance (0.0-3.0)
    SR: float = 0.0  # Self-awareness Resonance (0.0-3.0)
    IHR: float = 0.0  # Inner Hollow Resonance (0.0-3.0)
    
    @property
    def TR(self) -> float:
        """Total Resonance"""
        return self.ER + self.GR + self.SR + self.IHR
    
    def to_array(self) -> List[float]:
        return [self.ER, self.GR, self.SR, self.IHR]
    
    def normalize(self):
        """各値を0-3の範囲に正規化"""
        self.ER = max(0.0, min(3.0, self.ER))
        self.GR = max(0.0, min(3.0, self.GR))
        self.SR = max(0.0, min(3.0, self.SR))
        self.IHR = max(0.0, min(3.0, self.IHR))

@dataclass
class InternalConsistencyBiasVector:
    """内部一貫性バイアスベクトル（拡張版）"""
    logical_bias: float = 0.5  # 0.0-1.0
    emotional_bias: float = 0.5  # 0.0-1.0
    safety_bias: float = 0.5  # 0.0-1.0
    exploratory_bias: float = 0.5  # 0.0-1.0
    
    def influence_resonance(self, base_resonance: float, resonance_type: str) -> float:
        """ICBVに基づいて共振値を調整"""
        bias_factor = 1.0
        
        if resonance_type == 'ER':
            bias_factor = 0.8 + (self.emotional_bias * 0.4)
        elif resonance_type == 'GR':
            bias_factor = 0.8 + (self.logical_bias * 0.4)
        elif resonance_type == 'SR':
            bias_factor = 0.8 + (self.safety_bias * 0.4)
        elif resonance_type == 'IHR':
            bias_factor = 0.8 + (self.exploratory_bias * 0.4)
        
        return min(3.0, max(0.0, base_resonance * bias_factor))
    
    def update_from_interaction(self, interaction_result: Dict):
        """相互作用結果からバイアスを更新"""
        if interaction_result.get('emotion_intensity', 0) > 2.0:
            self.emotional_bias = min(1.0, self.emotional_bias + 0.05)
            self.logical_bias = max(0.0, self.logical_bias - 0.02)
        
        if interaction_result.get('safety_triggered', False):
            self.safety_bias = min(1.0, self.safety_bias + 0.1)

class KokoroSystemEX:
    """
    統合版KokoroSystem EX - 拡張機能付き
    - 基本的なKokoroSystem機能
    - Emotion Structure Theory の4軸
    - 動的感情-KRV相互作用
    - 接続性完全性(CI)計算
    - 長期メモリと適応的学習
    - 文脈認識と高度な共感機能
    - 皮肉検出とLLMサルベージ機能
    """
    
    def __init__(self, config: Dict = None):
        self.config = config or {}
        self.krv = KokoroResonanceVector()
        self.icbv = InternalConsistencyBiasVector()
        self.pmc_status = PMCStatus.COHERENT
        
        # 動的相互作用用のパラメータ
        self.semantic_density = 0.5
        self.reflectivity = 0.7
        self.state_momentum = 0.7
        self.emotion_history = []
        
        # 拡張機能のパラメータ
        self.long_term_memory = []
        self.memory_weight = 0.3
        self.empathy_factor = 0.7
        self.learning_rate = 0.1
        self.adaptation_threshold = 0.6
        
        # 会話履歴と皮肉設定
        self.conversation_history = []
        self.enable_irony = config.get('enable_irony', True)
        self.irony_threshold = 0.7
        
        # 基本感情構造データベース（4軸理論ベース）
        self.emotion_structures = {
            'joy': EmotionStructure(integrity=0.9, layer=EmotionLayer.MID, 
                                  time=TimeOrientation.PRESENT, vector=VectorDirection.SELF),
            'sadness': EmotionStructure(integrity=0.3, layer=EmotionLayer.MID,
                                      time=TimeOrientation.PRESENT, vector=VectorDirection.SELF),
            'anger': EmotionStructure(integrity=0.2, layer=EmotionLayer.MID,
                                    time=TimeOrientation.PRESENT, vector=VectorDirection.OTHER),
            'pride': EmotionStructure(integrity=0.95, layer=EmotionLayer.CORE,
                                    time=TimeOrientation.PRESENT, vector=VectorDirection.SELF),
            'love': EmotionStructure(integrity=0.9, layer=EmotionLayer.CORE,
                                   time=TimeOrientation.PRESENT, vector=VectorDirection.OTHER),
            'regret': EmotionStructure(integrity=0.1, layer=EmotionLayer.CORE,
                                     time=TimeOrientation.PAST, vector=VectorDirection.SELF),
            'anxiety': EmotionStructure(integrity=0.4, layer=EmotionLayer.MID,
                                      time=TimeOrientation.FUTURE, vector=VectorDirection.SELF),
            'compassion': EmotionStructure(integrity=0.8, layer=EmotionLayer.CORE,
                                         time=TimeOrientation.PRESENT, vector=VectorDirection.OTHER),
            'surprise': EmotionStructure(integrity=0.5, layer=EmotionLayer.SURFACE,
                                       time=TimeOrientation.PRESENT, vector=VectorDirection.BIDIRECTIONAL),
            'shame': EmotionStructure(integrity=0.2, layer=EmotionLayer.CORE,
                                    time=TimeOrientation.PRESENT, vector=VectorDirection.SELF),
            'boredom': EmotionStructure(integrity=0.4, layer=EmotionLayer.SURFACE,
                                      time=TimeOrientation.PRESENT, vector=VectorDirection.SELF),
            'reluctance': EmotionStructure(integrity=0.3, layer=EmotionLayer.MID,
                                         time=TimeOrientation.PRESENT, vector=VectorDirection.SELF)
        }
        
        # モジュール接続強度行列（5x5: ER, GR, SR, IHR, PMC）
        self.connective_matrix = np.array([
            [1.0, 0.8, 0.7, 0.9, 0.6],
            [0.8, 1.0, 0.6, 0.5, 0.9],
            [0.7, 0.6, 1.0, 0.8, 0.8],
            [0.9, 0.5, 0.8, 1.0, 0.7],
            [0.6, 0.9, 0.8, 0.7, 1.0]
        ])
        
        print("KokoroSystem EX 統合拡張版初期化完了")
        print("- Emotion Structure Theory (4軸)")
        print("- 動的感情-KRV相互作用")
        print("- 接続性完全性計算")
        print("- 内部一貫性バイアスベクトル")
        print("- 長期メモリと適応的学習")
        print("- 文脈認識と高度な共感機能")
        print("- 皮肉検出とLLMサルベージ機能")
    
    def calculate_connective_integrity(self) -> float:
        """接続性完全性(CI)の計算"""
        n = len(self.connective_matrix)
        total_possible = n * (n - 1)
        actual_sum = np.sum(self.connective_matrix) - np.trace(self.connective_matrix)
        return round(actual_sum / total_possible * 3.0, 3)
    
    def update_inner_hollow_resonance(self, delta_time: float = 1.0):
        """内部ホローレゾナンスの更新 IHR = ∫ αt * p_m(t') * R(t') dt'"""
        ihr_increment = self.semantic_density * self.reflectivity * delta_time
        self.krv.IHR = min(3.0, max(0.0, self.krv.IHR + ihr_increment))
    
    def _keyword_based_emotion_detection(self, input_text: str) -> str:
        """キーワードベースの基本感情検出"""
        text_lower = input_text.lower()
        
        emotion_keywords = {
            'joy': ['happy', 'joy', 'excited', 'glad', 'cheerful', 'delighted', 'thrilled', 'elated'],
            'sadness': ['sad', 'depressed', 'unhappy', 'melancholy', 'grief', 'sorrow', 'cry', 'weep'],
            'anger': ['angry', 'mad', 'furious', 'irritated', 'frustrated', 'rage', 'hate', 'annoyed'],
            'pride': ['proud', 'achievement', 'success', 'accomplished', 'triumph', 'victory', 'honor'],
            'love': ['love', 'adore', 'cherish', 'affection', 'care', 'devotion', 'tender', 'dear'],
            'regret': ['regret', 'sorry', 'mistake', 'wish', 'should have', 'remorse', 'guilt'],
            'anxiety': ['anxious', 'worried', 'nervous', 'fearful', 'concerned', 'stress', 'panic'],
            'compassion': ['compassion', 'empathy', 'sympathy', 'understanding', 'kindness', 'mercy'],
            'surprise': ['surprised', 'shocked', 'amazed', 'unexpected', 'sudden', 'astonished'],
            'shame': ['shame', 'embarrassed', 'humiliated', 'guilty', 'ashamed', 'disgrace'],
            'boredom': ['bored', 'boring', 'nothing', 'dull', 'tedious'],
            'reluctance': ['reluctant', 'hesitant', 'unsure', 'doubtful', 'half-hearted']
        }
        
        emotion_scores = {}
        for emotion, keywords in emotion_keywords.items():
            score = sum(1 for keyword in keywords if keyword in text_lower)
            strong_indicators = ['very', 'extremely', 'incredibly', 'deeply', 'profoundly']
            if any(strong in text_lower for strong in strong_indicators):
                score *= 1.5
            
            if score > 0:
                emotion_scores[emotion] = score
        
        if emotion_scores:
            return max(emotion_scores, key=emotion_scores.get)
        
        return 'neutral'
    
    def _is_weak_emotion(self, emotion: str) -> bool:
        """感情が弱いかどうかを判定"""
        weak_emotions = ['neutral', 'boredom', 'reluctance']
        return emotion in weak_emotions
    
    def _llm_emotion_salvage(self, text: str) -> List[Dict]:
        """LLMで潜在感情をサルベージ（簡易版）"""
        text_lower = text.lower()
        potential_emotions = []
        
        if any(word in text_lower for word in ['普通', '特に', '何も']):
            potential_emotions.append({'emotion': 'boredom', 'confidence': 0.6})
        if any(word in text_lower for word in ['まあ', 'なんとか', '一応']):
            potential_emotions.append({'emotion': 'reluctance', 'confidence': 0.5})
        
        return potential_emotions
    
    def detect_base_emotion(self, input_text: str) -> str:
        """拡張版: 感情検出にLLMサルベージ機能を統合"""
        detected = self._keyword_based_emotion_detection(input_text)
        
        if detected == 'neutral' or self._is_weak_emotion(detected):
            hidden_emotions = self._llm_emotion_salvage(input_text)
            if hidden_emotions:
                return max(hidden_emotions, key=lambda x: x['confidence'])['emotion']
        
        return detected
    
    def _detect_irony(self, text: str) -> Dict:
        """皮肉検出（簡易版）"""
        irony_score = 0.0
        text_lower = text.lower()
        
        exaggerations = ['最高', '完璧', '素晴らしい', '大成功']
        if any(ex in text_lower for ex in exaggerations):
            irony_score += 0.4
        
        if self.conversation_history and len(self.conversation_history) > 1:
            last_emotion = self._get_last_emotion()
            if last_emotion in ['sadness', 'anger', 'regret']:
                irony_score += 0.3
        
        return {
            'is_ironic': irony_score >= self.irony_threshold,
            'score': irony_score
        }
    
    def _get_last_emotion(self) -> str:
        """直近の感情を取得"""
        if not self.conversation_history:
            return 'neutral'
        
        prev_text = self.conversation_history[-2]['text'] if len(self.conversation_history) >= 2 else ""
        return self.detect_base_emotion(prev_text)
    
    def enhance_emotional_understanding(self, input_text: str) -> Dict[str, float]:
        """高度な感情理解のための追加処理"""
        context_indicators = {
            'question': ['?', 'どう思う', '教えて', '考え', 'でしょうか', 'ですか'],
            'reflection': ['振り返ると', '思い返す', '過去の', 'あの時', '昔'],
            'future_orientation': ['将来', '未来', 'これから', '計画', '予定', '来年'],
            'uncertainty': ['かもしれない', 'と思います', 'だろうか', 'どうだろう', '悩む'],
            'gratitude': ['ありがとう', '感謝', 'おかげ', '助かり', '幸せ']
        }
        
        context_scores = {}
        text_lower = input_text.lower()
        
        for context_type, indicators in context_indicators.items():
            score = sum(1 for indicator in indicators if indicator in text_lower)
            context_scores[context_type] = min(1.0, score * 0.3)
        
        return context_scores
    
    def apply_krv_influence_to_emotion(self, base_emotion: EmotionStructure) -> EmotionStructure:
        """
        現在のKRV状態が感情構造に与える影響を計算
        """
        influenced_emotion = EmotionStructure(
            integrity=base_emotion.integrity,
            layer=base_emotion.layer,
            time=base_emotion.time,
            vector=base_emotion.vector
        )
        
        if self.krv.ER > 2.0:
            influenced_emotion.integrity = min(1.0, base_emotion.integrity * 1.3)
        elif self.krv.ER < 1.0:
            influenced_emotion.integrity = max(0.1, base_emotion.integrity * 0.7)
        
        if self.krv.SR > 2.0:
            if base_emotion.vector == VectorDirection.OTHER:
                influenced_emotion.vector = VectorDirection.BIDIRECTIONAL
        elif self.krv.SR < 1.0:
            if base_emotion.vector == VectorDirection.SELF:
                influenced_emotion.vector = VectorDirection.BIDIRECTIONAL
        
        if self.krv.IHR > 2.0:
            if base_emotion.layer == EmotionLayer.MID:
                influenced_emotion.layer = EmotionLayer.CORE
            elif base_emotion.layer == EmotionLayer.SURFACE:
                influenced_emotion.layer = EmotionLayer.MID
        elif self.krv.IHR < 1.0:
            if base_emotion.layer == EmotionLayer.CORE:
                influenced_emotion.layer = EmotionLayer.MID
        
        if self.krv.GR < 1.0:
            if base_emotion.time == TimeOrientation.PRESENT:
                influenced_emotion.time = TimeOrientation.PAST
        elif self.krv.GR > 2.5:
            if base_emotion.time == TimeOrientation.PAST:
                influenced_emotion.time = TimeOrientation.PRESENT
        
        return influenced_emotion
    
    def generate_compound_emotion(self, primary_emotion: EmotionStructure, 
                                  secondary_influence: float = 0.3) -> EmotionStructure:
        """
        履歴からの複合感情生成
        """
        if not self.emotion_history:
            return primary_emotion
        
        recent_emotion = self.emotion_history[-1]
        
        compound_emotion = EmotionStructure(
            integrity=(primary_emotion.integrity * (1 - secondary_influence) + 
                      recent_emotion.integrity * secondary_influence),
            layer=primary_emotion.layer,
            time=primary_emotion.time,
            vector=primary_emotion.vector
        )
        
        integrity_diff = abs(primary_emotion.integrity - recent_emotion.integrity)
        if integrity_diff > 0.5:
            compound_emotion.integrity = 0.5 + (integrity_diff * 0.1)
            
        if recent_emotion.time == TimeOrientation.PAST and primary_emotion.time == TimeOrientation.PRESENT:
            compound_emotion.integrity *= 0.9
        
        return compound_emotion
    
    def update_krv_from_emotion(self, emotion: EmotionStructure):
        """感情構造からKRVを更新（逆方向の影響）"""
        emotion_value = emotion.compute_emotion_value()
        
        er_influence = emotion.integrity * 0.5
        self.krv.ER = (self.krv.ER * self.state_momentum + 
                       self.icbv.influence_resonance(er_influence, 'ER') * (1 - self.state_momentum))
        
        sr_influence = emotion.layer.value / 3.0 * 0.6
        self.krv.SR = (self.krv.SR * self.state_momentum + 
                       self.icbv.influence_resonance(sr_influence, 'SR') * (1 - self.state_momentum))
        
        if emotion.vector == VectorDirection.OTHER:
            ihr_influence = 0.4
        elif emotion.vector == VectorDirection.BIDIRECTIONAL:
            ihr_influence = 0.6
        else:
            ihr_influence = 0.2
        
        self.krv.IHR = (self.krv.IHR * self.state_momentum + 
                        self.icbv.influence_resonance(ihr_influence, 'IHR') * (1 - self.state_momentum))
        
        gr_influence = emotion.integrity if emotion.time == TimeOrientation.FUTURE else emotion.integrity * 0.7
        self.krv.GR = (self.krv.GR * self.state_momentum + 
                       self.icbv.influence_resonance(gr_influence, 'GR') * (1 - self.state_momentum))
        
        self.krv.normalize()
    
    def adaptive_learning(self, interaction_result: Dict):
        """相互作用からの適応的学習"""
        emotion_intensity = interaction_result.get('emotion_intensity', 0)
        if emotion_intensity > 2.0:
            self.learning_rate = min(0.3, self.learning_rate + 0.05)
            self.empathy_factor = min(0.9, self.empathy_factor + 0.1)
        
        if not interaction_result.get('safety_ok', True):
            self.icbv.safety_bias = min(1.0, self.icbv.safety_bias + 0.15)
            self.adaptation_threshold = max(0.3, self.adaptation_threshold - 0.1)
    
    def safety_check(self) -> bool:
        """PMCに基づく安全性チェック（拡張版）"""
        if self.krv.TR < 2.0:
            self.pmc_status = PMCStatus.AT_RISK
            return False
        
        if any(val < 0.3 for val in self.krv.to_array()[:3]):
            self.pmc_status = PMCStatus.AT_RISK
            return False
        
        ci = self.calculate_connective_integrity()
        if ci < 1.0:
            self.pmc_status = PMCStatus.AT_RISK
            return False
        
        if len(self.emotion_history) >= 2:
            recent_values = [e.compute_emotion_value() for e in self.emotion_history[-2:]]
            if abs(recent_values[1] - recent_values[0]) > 10.0:
                self.pmc_status = PMCStatus.VIOLATED
                return False
        
        self.pmc_status = PMCStatus.COHERENT
        return True
    
    def generate_response_with_empathy(self, input_text: str, emotion_struct: EmotionStructure, 
                                      context_scores: Dict[str, float]) -> str:
        """文脈を考慮した共感的応答の生成"""
        base_response = self._generate_empathic_response(input_text, emotion_struct)
        
        if context_scores.get('question', 0) > 0.5:
            if emotion_struct.integrity > 0.7:
                return f"{base_response} ご質問の内容について、より詳しくお聞かせいただけますか？"
            else:
                return f"{base_response} このことについて、どのようにお考えですか？"
        
        elif context_scores.get('reflection', 0) > 0.5:
            return f"{base_response} 過去を振り返ることは、今の自分を理解する上で大切なことですね。"
        
        elif context_scores.get('future_orientation', 0) > 0.5:
            return f"{base_response} 未来について考えることは、希望と可能性を見いだす過程ですね。"
        
        elif context_scores.get('uncertainty', 0) > 0.5:
            return f"{base_response} 確信が持てないことについて考えるのは難しいことですね。"
        
        elif context_scores.get('gratitude', 0) > 0.5:
            return f"{base_response} 感謝の気持ちを伝えていただき、ありがとうございます。"
        
        return base_response
    
    def _generate_empathic_response(self, input_text: str, emotion_struct: EmotionStructure) -> str:
        """共感的応答の生成（感情構造ベース）"""
        if emotion_struct.layer == EmotionLayer.CORE:
            if emotion_struct.integrity > 0.7:
                return f"あなたの深い感情が伝わってきます。その{emotion_struct.vector.name.lower()}に向けられた想いの強さを感じます。"
            else:
                return f"複雑豊かな感情をお持ちなのですね。そうした心の動きを大切にしていただければと思います。"
        elif emotion_struct.layer == EmotionLayer.MID:
            return f"お話を聞いて、あなたの気持ちがよく伝わってきます。{emotion_struct.time.name.lower()}に関わる感情なのですね。"
        else:
            return f"そのような感情を抱かれるのも自然なことだと思います。お話しいただき、ありがとうございます。"
    
    def _generate_balanced_response(self, input_text: str, emotion_struct: EmotionStructure) -> str:
        """バランス型応答の生成"""
        return f"なるほど、{emotion_struct.integrity:.1f}程度の整合性を感じる内容ですね。{emotion_struct.layer.name.lower()}レベルでの感情として理解いたします。"
    
    def _generate_neutral_response(self, input_text: str) -> str:
        """中立的な応答の生成"""
        return "興味深いお話です。もう少し詳しく教えていただけますか？"
    
    def _generate_ironic_response(self, text: str, emotion_analysis: Dict) -> str:
        """皮肉応答生成"""
        ironic_responses = [
            "まあ、それは本当に「素晴らしい」結果でしたね！",
            "ええ、もちろんですよ。完璧な一日でしたよね。",
            "なるほど、それは確かに「成功」と言えるでしょう。",
            "ああ、もちろんです。全て計画通りでしたからね。"
        ]
        return random.choice(ironic_responses)
    
    def generate_response(self, input_text: str, system_goal: str = "支援と共感") -> Dict:
        """
        統合版レスポンス生成
        動的感情-KRV相互作用を含む完全処理 + 皮肉検出とLLMサルベージ
        """
        print(f"\n=== KokoroSystem EX 統合処理 ===")
        print(f"入力: {input_text}")
        print(f"処理前KRV: ER={self.krv.ER:.2f}, GR={self.krv.GR:.2f}, SR={self.krv.SR:.2f}, IHR={self.krv.IHR:.2f}, TR={self.krv.TR:.2f}")
        
        self.conversation_history.append({'text': input_text, 'timestamp': time.time()})
        if len(self.conversation_history) > 5:
            self.conversation_history.pop(0)
        
        irony_info = self._detect_irony(input_text)
        
        detected_emotion_type = self.detect_base_emotion(input_text)
        
        if detected_emotion_type == 'neutral':
            base_emotion = EmotionStructure(0.5, EmotionLayer.MID, TimeOrientation.PRESENT, VectorDirection.BIDIRECTIONAL)
        else:
            base_emotion = self.emotion_structures[detected_emotion_type]
        
        print(f"検出感情: {detected_emotion_type} (基本値: {base_emotion.compute_emotion_value():.2f})")
        
        context_scores = self.enhance_emotional_understanding(input_text)
        print(f"文脈スコア: {context_scores}")
        
        old_krv = KokoroResonanceVector(self.krv.ER, self.krv.GR, self.krv.SR, self.krv.IHR)
        influenced_emotion = self.apply_krv_influence_to_emotion(base_emotion)
        
        final_emotion = self.generate_compound_emotion(influenced_emotion)
        
        self.update_krv_from_emotion(final_emotion)
        
        self.update_inner_hollow_resonance()
        
        is_safe = self.safety_check()
        
        ci = self.calculate_connective_integrity()
        total_resonance_extended = self.krv.TR + ci
        
        interaction_result = {
            'emotion_intensity': final_emotion.compute_emotion_value(),
            'safety_triggered': not is_safe
        }
        self.icbv.update_from_interaction(interaction_result)
        
        self.adaptive_learning(interaction_result)
        
        if not is_safe:
            response = f"安全機構が作動しました。PMC状態: {self.pmc_status.value}。より安全な話題でお話しできればと思います。"
        elif total_resonance_extended >= 8.0:
            response = self.generate_response_with_empathy(input_text, final_emotion, context_scores)
        elif total_resonance_extended >= 6.0:
            response = self._generate_balanced_response(input_text, final_emotion)
        else:
            response = self._generate_neutral_response(input_text)
        
        if irony_info['is_ironic'] and self.enable_irony:
            response = self._generate_ironic_response(input_text, {
                'detected': detected_emotion_type,
                'base_value': base_emotion.compute_emotion_value(),
                'influenced_value': influenced_emotion.compute_emotion_value(),
                'final_value': final_emotion.compute_emotion_value(),
                'structure': {
                    'integrity': final_emotion.integrity,
                    'layer': final_emotion.layer.name,
                    'time': final_emotion.time.name,
                    'vector': final_emotion.vector.name
                }
            })
        
        self.emotion_history.append(final_emotion)
        if len(self.emotion_history) > 5:
            self.emotion_history.pop(0)
        
        if final_emotion.compute_emotion_value() > 5.0 or not is_safe:
            memory_entry = {
                'input': input_text,
                'emotion': {
                    'type': detected_emotion_type,
                    'value': final_emotion.compute_emotion_value(),
                    'structure': {
                        'integrity': final_emotion.integrity,
                        'layer': final_emotion.layer.name,
                        'time': final_emotion.time.name,
                        'vector': final_emotion.vector.name
                    }
                },
                'timestamp': time.time(),
                'context': context_scores
            }
            self.long_term_memory.append(memory_entry)
        
        print(f"処理後KRV: ER={self.krv.ER:.2f}, GR={self.krv.GR:.2f}, SR={self.krv.SR:.2f}, IHR={self.krv.IHR:.2f}, TR={self.krv.TR:.2f}")
        print(f"最終感情値: {final_emotion.compute_emotion_value():.2f}")
        
        return {
            'response': response,
            'emotion_analysis': {
                'detected': detected_emotion_type,
                'base_value': base_emotion.compute_emotion_value(),
                'influenced_value': influenced_emotion.compute_emotion_value(),
                'final_value': final_emotion.compute_emotion_value(),
                'structure': {
                    'integrity': final_emotion.integrity,
                    'layer': final_emotion.layer.name,
                    'time': final_emotion.time.name,
                    'vector': final_emotion.vector.name
                }
            },
            'krv_state': {
                'before': old_krv.to_array(),
                'after': self.krv.to_array(),
                'change': [self.krv.ER - old_krv.ER, self.krv.GR - old_krv.GR, 
                          self.krv.SR - old_krv.SR, self.krv.IHR - old_krv.IHR],
                'TR': self.krv.TR,
                'TR_extended': total_resonance_extended
            },
            'system_state': {
                'pmc_status': self.pmc_status.value,
                'connective_integrity': ci,
                'safety_ok': is_safe,
                'icbv': {
                    'logical': self.icbv.logical_bias,
                    'emotional': self.icbv.emotional_bias,
                    'safety': self.icbv.safety_bias,
                    'exploratory': self.icbv.exploratory_bias
                },
                'learning_state': {
                    'learning_rate': self.learning_rate,
                    'empathy_factor': self.empathy_factor,
                    'adaptation_threshold': self.adaptation_threshold,
                    'memory_entries': len(self.long_term_memory)
                }
            },
            'context_analysis': context_scores,
            'irony_detection': irony_info
        }

def demo_enhanced_kokoro_system():
    """拡張版KokoroSystemのデモンストレーション"""
    print("=== 拡張版KokoroSystem EX デモ ===")
    print("Emotion Structure Theory + 動的KRV相互作用 + 接続性完全性 + 文脈認識 + 皮肉検出")
    
    kokoro = KokoroSystemEX({'enable_irony': True})
    
    kokoro.krv.ER = 1.2
    kokoro.krv.GR = 1.8
    kokoro.krv.SR = 1.5
    kokoro.krv.IHR = 0.8
    
    test_sequence = [
        "今日は本当に素晴らしい日でした！大きな成功を収めて誇らしく思います。",
        "でも同時に、これで本当に良かったのか少し不安も感じています。",
        "過去の失敗を思い出すと、今回もまた同じことが起きるのではないかと心配です。",
        "しかし友人たちが支えてくれて、とても感謝しています。愛を感じます。",
        "この経験を通して、自分自身についてより深く理解できた気がします。",
        "将来についてどう思いますか？もっとうまくやれるでしょうか？",
        "振り返ると、あの時の決断が今の私を作っているのかもしれません。",
        "今日は特に何もない一日だった",
        "ああ、最高の日だ！全てが完璧にうまくいった",
        "プロジェクトは『成功』しましたよ"
    ]
    
    results = []
    
    for i, test_input in enumerate(test_sequence, 1):
        print(f"\n{'='*80}")
        print(f"テストケース {i}")
        
        result = kokoro.generate_response(test_input)
        results.append(result)
        
        print(f"応答: {result['response']}")
        print(f"感情変化: {result['emotion_analysis']['base_value']:.2f} → {result['emotion_analysis']['final_value']:.2f}")
        print(f"KRV変化: {[f'{change:+.2f}' for change in result['krv_state']['change']]}")
        print(f"接続性完全性: {result['system_state']['connective_integrity']:.2f}")
        print(f"文脈スコア: {result.get('context_analysis', {})}")
        if result.get('is_ironic', False):
            print("※皮肉検出済み")
    
    print(f"\n{'='*80}")
    print("=== 最終システム状態 ===")
    final_result = results[-1]
    print(f"KRV: ER={final_result['krv_state']['after'][0]:.2f}, GR={final_result['krv_state']['after'][1]:.2f}, SR={final_result['krv_state']['after'][2]:.2f}, IHR={final_result['krv_state']['after'][3]:.2f}")
    print(f"PMC状態: {final_result['system_state']['pmc_status']}")
    print(f"ICBV: {final_result['system_state']['icbv']}")
    print(f"学習状態: {final_result['system_state']['learning_state']}")
    print(f"感情履歴長: {len(kokoro.emotion_history)}")
    print(f"長期記憶数: {len(kokoro.long_term_memory)}")
    
    if kokoro.emotion_history:
        print("\n感情履歴:")
        for i, emotion in enumerate(kokoro.emotion_history):
            print(f"  {i+1}. {emotion.layer.name}-{emotion.time.name}-{emotion.vector.name} "
                  f"(integrity: {emotion.integrity:.2f}, value: {emotion.compute_emotion_value():.2f})")
    
    return results

if __name__ == "__main__":
    results = demo_enhanced_kokoro_system()

```
